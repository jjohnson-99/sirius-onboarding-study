# `pipeline/sirius_pipeline_converter.cpp` → Pipeline-Construction Map (Step 4 core)

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), the
**Step-4 / pipeline-construction** reading (added after this file was found to be the gap where
Step 4 actually lives). Tagged **study** — at 1,343 lines it's the biggest file in
`src/pipeline/` and where most of `docs/super-sirius/execution-flow.md` **Step 4** (sub-steps
4b/4c/4d) is implemented. Read that doc's Step 4 alongside, and read
[`sirius_meta_pipeline.md`](sirius_meta_pipeline.md) (Step 4a) and
[`sirius_pipeline.md`](sirius_pipeline.md) first — this file consumes the former and produces the
latter.

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**; study-note
> links are relative to this file. Line numbers accurate as of 2026-06-17 — re-grep if moved.

## Why this file matters (and why you couldn't find Step 4)

`execution-flow.md` headlines Step 4 as `sirius_engine.cpp::initialize_internal()`, but that
function is a ~50-line **orchestrator** ([`../sirius_engine.md`](../sirius_engine.md)). The actual
construction is delegated:

```
initialize_internal()                    sirius_engine.cpp:350
├─ root.build() / root.ready()           ─▶ sirius_meta_pipeline.cpp   · Step 4a
├─ converter.convert(root)               ─▶ THIS FILE                  · Step 4b + 4d
└─ materialize_repository_wiring(result) ─▶ repository_wiring_materializer.cpp · Step 4c
```

So **this file is where the operator splitting referenced all over Weeks 2–3 is born** — the
local→merge aggregate pairs, the `PARTITION`/`CONCAT` around joins, the 4-phase sort. If a map
ever said "the engine creates the pair…", the *real* site is here
(`construct_sirius_specific_operator`, cpp 61, + the `split_*` methods).

## The key idea first: meta-pipelines in → execution pipelines out, via a 7-phase `convert()`

The meta-pipeline tree (Step 4a) is DuckDB-shaped: source→…→sink chains with build-side
children. The converter turns it into **Sirius execution pipelines** by deep-copying, **splitting**
operators into Sirius multi-pipeline shapes, emitting topological wiring descriptors, and
finalizing dependencies. The whole thing is one public method, `convert()` (cpp 123), calling
seven private phases in order — each reads/mutates three member accumulators (`scheduled_`,
`inserted_operators_`, `repository_wirings_`).

## Call sequence

`─▶` leaves this file; `· Step N` = `execution-flow.md` step.

```
convert(root_meta_pipeline)                         :123   ◀─ sirius_engine::initialize_internal :380
├─ schedule_and_copy_pipelines(root)                :143   Phase 1: topo-order + deep-copy  → copied_scheduled
│     └─ (operator factory used later) construct_sirius_specific_operator(op)  :61
├─ split_pipelines(copied_scheduled)                :923   Phase 2: per-pipeline operator splitting ─┐
│   ├─ split_table_scan_source       :360  TABLE_SCAN → GPU scan op (parquet / duckdb-native)        │
│   │     ├─ insert_parquet_scan_operator         :237                                               │
│   │     └─ insert_duckdb_native_scan_operator    :283                                              │
│   ├─ split_cpu_source              :385  COLUMN_DATA_SCAN/EMPTY_RESULT/DUMMY_SCAN → CPU_SOURCE      │ · Step 4b
│   ├─ split_intermediate_joins      :429  mid-pipeline joins → PARTITION+CONCAT breakers            │
│   └─ dispatch on sink type (mutually exclusive):                                                   │
│        ├─ split_join_sink             :547  HASH_JOIN/NLJ → PARTITION + CONCAT (build side)         │
│        ├─ split_group_aggregate_sink  :624  HASH_GROUP_BY → PARTITION + MERGE_GROUP_BY             │
│        ├─ split_order_by_sink         :686  ORDER_BY → SORT_SAMPLE → SORT_PARTITION → MERGE_SORT    │
│        ├─ split_top_n_sink            :780  TOP_N → MERGE_TOP_N                                     │
│        └─ split_delim_join_sink       :813  DELIM_JOIN → partition_join + distinct branches         │
├─ compute_repository_wiring()                       :960   Phase 3: assign pipeline ids + emit wiring · Step 4c
├─ setup_pipeline_parents()                          :1103  Phase 4: parents from wiring descriptors
├─ finalize_pipeline_structure()                     :1119  Phase 5: merge sink into operators[] · Step 4d
├─ link_join_partition_siblings()                    :1134  Phase 6: link build/probe PARTITION pair
└─ configure_partition_min_partitions()              :1176  Phase 7: multi-GPU partition floor
returns pipeline_conversion_result ─▶ materialize_repository_wiring(...)  · Step 4c (runtime half)
```

## Phase 1 — `schedule_and_copy_pipelines` (cpp 143): topo-order, then deep-copy

Two jobs in one method:

1. **Topological scheduling.** It flattens every meta-pipeline (`get_meta_pipelines`, cpp 150),
   sets `meta_pipeline_count_` (= number of `PipelineCompleteEvent`s), then loops with a
   round-robin `meta` cursor (cpp 159–204): a meta-pipeline's pipelines are appended to
   `sirius_scheduled` only once **all its child meta-pipelines and all its base pipeline's
   dependencies are already scheduled** (cpp 170–183). This produces a dependency-respecting
   order (build pipelines before the probes that need them).
   - **Subtlety:** a pipeline whose source is a `HASH_JOIN` is *skipped* here (cpp 189–197) —
     the hash-join-as-source pipeline (FULL OUTER scan of the HT) is wired in later; the
     right-join branch even has a commented-out `MODIFIED_PIPELINE` line (dead experiment).
2. **Deep copy.** Each scheduled pipeline is copied into a fresh `sirius_pipeline` (cpp 207–219)
   with its `source` / `operators` / `sink` reassigned. **Note what's copied:** the *pipeline
   container* is new, but `operators` holds `reference_wrapper`s — the copy **shares the same
   operator objects** with the plan. The converter owns a mutable *pipeline structure*, not new
   operators. Returns `copied_scheduled` (Phase 2's input).

## Phase 2 — splitting (the heart, Step 4b)

`split_pipelines` (cpp 923) loops `copied_scheduled` and, per pipeline, runs three preprocessing
passes (`split_table_scan_source`, `split_cpu_source`, `split_intermediate_joins`) then a
**mutually-exclusive dispatch on the sink type** (cpp 939–956). Each `split_*` rewrites one
pipeline into a Sirius multi-pipeline shape, pushing breaker operators into `inserted_operators_`
(owned by the result) and new pipelines into `scheduled_`. Anything with an unhandled sink is
appended unchanged (cpp 955).

### The recurring "split into local + merge, then rewire downstream" pattern

Several splits (group-agg, order-by, top-n, delim-distinct) share one shape:

1. keep the original (**local**) operator as the current pipeline's sink (per-batch partial);
2. mint a **merge** operator via `construct_sirius_specific_operator` (cpp 61);
3. (for grouped agg / joins) insert a `PARTITION` between them;
4. add a merge pipeline `… → MERGE`;
5. **rewire downstream:** for every later pipeline whose `source` was the local op, repoint it to
   the **merge** op (e.g. cpp 659–663) — because the merge is the true producer of the final
   result that downstream consumes.

That step-5 loop (`for j = pipeline_idx+1 … : if source == old_op → merge_op`) is the thing to
internalize; it appears in every "two-phase" split.

### Scan sources — `split_table_scan_source` (cpp 360)

`TABLE_SCAN` is replaced by a GPU scan operator inserted at the front of `operators`
(`finalize_pipeline_structure` later makes it the source):
- **parquet** (`insert_parquet_scan_operator`, cpp 237) — builds a `parquet_ingestible_table_info`
  (resolved file paths from `MultiFileBindData`, `table_filters` **moved** in for row-group
  pruning, hive partition indices, batch size) and a `sirius_gpu_scan_operator`. The companion
  metadata-scan op is parked on the scan op (driven later by the scan_manager on its own pool).
- **duckdb-native `seq_scan`** (`insert_duckdb_native_scan_operator`, cpp 283) — builds a
  `duckdb_native_ingestible_table_info` (storage + `client_context_`, projected columns/types,
  copied `table_filters`); base tables only.
- anything else throws (the legacy `duckdb_scan`/`iceberg_scan` task path was removed).

### `split_cpu_source` (cpp 385)

`EMPTY_RESULT` / `DUMMY_SCAN` / non-null `COLUMN_DATA_SCAN` get a `CPU_SOURCE` op fronting a new
single-op source pipeline. (A *null* `COLUMN_DATA_SCAN` is a LEFT_DELIM_JOIN's cached-chunk scan,
filled at runtime — splitting it would double-create a repository, so it's left alone, cpp 389–400.)

### `split_intermediate_joins` (cpp 429)

For each `HASH_JOIN`/`NLJ` sitting **mid-pipeline** (not the sink), insert a `PARTITION`+`CONCAT`
breaker and carve the pipeline at each join: the upstream segment becomes its own pipeline ending
at the op before the join, then `… → PARTITION → CONCAT → (rest)`. Handles multiple joins by
chaining (`prev_concat_ptr` feeds the next segment's source).

### `split_join_sink` (cpp 547) — the build side becomes PARTITION→CONCAT

When a pipeline's **sink** is a `HASH_JOIN`/`NLJ`, that pipeline is the **build side**. It's split
into: the upstream pipeline ending at the last pre-join op (as sink), a `PARTITION` pipeline, and a
`CONCAT` pipeline (`is_build = true`). The probe side and the join-as-source are separate pipelines
wired in `link_join_partition_siblings`. (Why partition+concat: hash-partition both sides so each
GPU joins matching partitions — see [hash join](../op/sirius_physical_hash_join.md) +
[partition](../op/sirius_physical_partition.md).)

### `split_group_aggregate_sink` (cpp 624)

`HASH_GROUP_BY` → keep it as the local sink, add `PARTITION`, add a `MERGE_GROUP_BY` pipeline, and
rewire downstream to the merge. `UNGROUPED_AGGREGATE` skips the PARTITION (no grouping keys) and
goes straight local → `MERGE_AGGREGATE`. See
[grouped](../op/sirius_physical_grouped_aggregate.md) / [ungrouped](../op/sirius_physical_ungrouped_aggregate.md).

### `split_order_by_sink` (cpp 686) — the 4-phase sort

`ORDER_BY` → four phases across three pipelines: (A) keep `ORDER_BY` as a per-batch local sort
(its projection is temporarily replaced with **identity** so sort keys survive into later phases,
cpp 697–708); (B) `ORDER_BY → SORT_SAMPLE → SORT_PARTITION` in one pipeline (partition reads
sample boundaries via `set_sample_op`, cpp 726); (C) `SORT_PARTITION → MERGE_SORT`, where
`MERGE_SORT` re-applies the original projection if it wasn't identity (cpp 752–758). Then rewire
downstream to `MERGE_SORT`. See [sorting](../../reference/explainers/sorting-and-order-by.md) /
[top_n map](../op/sirius_physical_top_n.md).

### `split_top_n_sink` (cpp 780) / `split_delim_join_sink` (cpp 813)

TOP_N → local `TOP_N` + `MERGE_TOP_N` (no partition). Delim joins are the most involved: a
`RIGHT_DELIM_JOIN` gets a `partition_join` + `CONCAT` on the join side **plus** a
`partition_distinct` + merge on the distinct side; `LEFT_DELIM_JOIN` wires its
`column_data_scan`. The composite sub-operators (`partition_join`, `distinct`, `column_data_scan`)
are stashed back on the delim-join op for the wiring phase.

## Phase 3 — `compute_repository_wiring` (cpp 960): assign ids, emit descriptors

Two things: (1) **assign `pipeline_id = index`** to every scheduled pipeline (cpp 972) — the
runtime materializer needs these to order ports deterministically; (2) build a
`source_to_pipelines` map and, per pipeline, **emit `repository_wiring` descriptors** (a plan-time
"sink→source edge": `port_id`, `barrier_type`, `source_op`, source/dest pipeline) via a local
`emit` lambda. It does **not** create repositories — that's the runtime half (below).

The barrier type is chosen per sink type (`MemoryBarrierType` is `{ PIPELINE, PARTIAL, FULL }`):

| Sink type | port | barrier | note |
|---|---|---|---|
| `MERGE_*` (group/sort/top_n/agg), CTE, generic intermediate sink | `default` | **FULL** | downstream waits for the full result |
| delim joins | `default` | **FULL** | wired via sub-ops (`partition_join`/`distinct`/`column_data_scan`) |
| `CONCAT` — build | `build` | **FULL** | dest resolved by finding the pipeline whose first op/sink is the `HASH_JOIN` (`get_parent_op`) |
| `CONCAT` — probe | `default` | **FULL** | dest via `source_to_pipelines` |
| `PARTITION`/`TOP_N`/`MERGE_SORT`/`SORT_PARTITION`/ungrouped | `default` | **PARTIAL if downstream is a CONCAT, else FULL** | concat can drain incrementally |
| `ORDER_BY` | `default` | **PIPELINE** | sample+partition consume batches as produced |
| `CPU_SOURCE` | `scan` | **PIPELINE** | streamed |
| `RESULT_COLLECTOR` | — | — | no wiring (terminal) |

### Plan-time descriptors vs. runtime materialize — the cleanest takeaway

`compute_repository_wiring` is **pure plan math**; nothing touches cuCascade. The runtime half is a
separate small file:

> **Folded-in file: `repository_wiring_materializer.cpp` (~65 lines).**
> `materialize_repository_wiring(wirings, data_repo_manager)` (called from `initialize_internal:383`,
> *after* `convert()` returns) turns each descriptor into a live `cucascade::shared_data_repository`
> + attaches an operator `port`. Special cases: `RIGHT_DELIM_JOIN` also gets a `FULL` port on its
> `partition_join` sibling; `LEFT_DELIM_JOIN` as a *destination* throws. **Precondition:** pipeline
> ids assigned first (here, in Phase 3), since ports are kept ordered by pipeline id.

(Port + barrier *semantics* live in [`../../weeks/week4-5-concepts.md`](../../weeks/week4-5-concepts.md)
Part 2 and `docs/super-sirius/data-management.md`.)

## Phases 4–7 — finalize

- **`setup_pipeline_parents` (cpp 1103)** — derive parent edges **from the wiring descriptors**
  (not from materialized ports, which don't exist until after `convert()` returns): clears each
  pipeline's `parents`/`dependencies`, then for each wiring pushes `dest` onto `source.parents`.
- **`finalize_pipeline_structure` (cpp 1119)** — **push each sink into `operators`** so
  `operators[]` is the full source→sink list, set `source = &operators[0]`, and populate each
  parent's `dependencies` from the (now-locked) parent links. *After this point the
  `source`/`sink` aliases are reliable* — the invariant the [task](gpu_pipeline_task.md) /
  [executor](gpu_pipeline_executor.md) rely on.
- **`link_join_partition_siblings` (cpp 1134)** — for each `HASH_JOIN`-as-source pipeline, walk
  `dependencies` to find build/probe `CONCAT`→`PARTITION` pairs (`deps[0]` = build concat→its
  `deps[0]` partition; `deps[1]` = probe concat→partition), **mutate the probe partition's wiring
  descriptor to PARTIAL** (cpp 1147–1152, since the port isn't materialized yet), and set the two
  partitions as each other's `_sibling_partition_op` (the link you met in the
  [partition map](../op/sirius_physical_partition.md)). A delim-join branch handles the
  `RIGHT_DELIM_JOIN` build partition.
- **`configure_partition_min_partitions` (cpp 1176)** — multi-GPU only (`num_gpus > 1`): force a
  partition-count floor of `num_gpus` on inputs above `num_gpus * 16 MiB` (`small_table_bytes`
  keeps tiny aggregations single-GPU). No-op on single GPU.

The result `pipeline_conversion_result` (`scheduled_pipelines`, `inserted_operators`,
`repository_wirings`, `meta_pipeline_count`) is moved back to the engine as `new_scheduled`.

## Outputs & members

| `pipeline_conversion_result` field (hpp 41) | What it is |
|---|---|
| `scheduled_pipelines` | the final ordered execution pipelines (with ids) |
| `inserted_operators` | the converter-minted breaker/merge/scan ops (ownership) |
| `repository_wirings` | plan-time wiring descriptors for the materializer |
| `meta_pipeline_count` | number of `PipelineCompleteEvent`s |

| Converter member | Role |
|---|---|
| `scheduled_` / `inserted_operators_` / `repository_wirings_` | the three accumulators built across phases |
| `build_ctx_` | plan-time config (`num_gpus`, `preserve_insertion_order`) |
| `op_params_` | tunables: `concat_batch_bytes`, `hash_partition_bytes`, `sort_sample_bytes`, `scan_task_batch_size`, … |
| `iceberg_cache_` | delete-data cache passed to the operator factory |
| `client_context_` | needed by the duckdb-native scan path |

## Types fundamental to *this* file

- **`pipeline_conversion_result`** *(hpp 41)* — the four outputs above. **Think:** "everything
  `initialize_internal` hands the scheduler."
- **`repository_wiring`** *(`repository_wiring.hpp`)* — one plan-time sink→source edge (`port_id`,
  `barrier_type`, `source_op`, source/dest pipeline). **Think:** "a wire on paper;
  `materialize_*` solders it."
- **`MemoryBarrierType {PIPELINE, PARTIAL, FULL}`** *(op)* — how much the consumer must wait:
  stream / drain-incrementally / wait-for-complete. **Think:** "the edge's backpressure setting."
- **`sirius_meta_pipeline`** *(input; [`sirius_meta_pipeline.md`](sirius_meta_pipeline.md))* — the
  DuckDB-shaped tree this converter consumes. **Think:** "Step 4a's output = this file's input."
- **`sirius_pipeline`** *(mapped in [`sirius_pipeline.md`](sirius_pipeline.md))* — the execution
  pipeline object the converter builds and rewires. **Think:** "the runnable unit the scheduler
  iterates."
- **`sirius_physical_partition` / `_concat`** *([partition map](../op/sirius_physical_partition.md))*
  — the breaker pair inserted around joins (hash-partition fan-out + re-concat). **Think:** "the
  Sirius shape a DuckDB join/group-by is rewritten into."

## Takeaway

This file *is* Step 4's substance: `convert()` topo-orders + deep-copies the meta-pipelines,
**splits** operators into Sirius's multi-pipeline shapes (the recurring local→PARTITION→merge
pattern, the 4-phase sort, the join breakers), assigns pipeline ids, emits **topological wiring
descriptors** with per-edge barrier types, then finalizes parents/operators and links the
build/probe partition siblings — all **pure plan math**, with cuCascade untouched until
`materialize_repository_wiring` runs back in the engine. With this mapped, the Step 4 trail no
longer goes cold at `initialize_internal`: 4a is [`sirius_meta_pipeline.md`](sirius_meta_pipeline.md),
4b/4d are here, 4c is the materializer above. Next, the constructed pipelines flow into the
scheduler tier ([`task_scheduler.md`](task_scheduler.md) →
[`gpu_pipeline_executor.md`](gpu_pipeline_executor.md), Steps 5/7).
