# `pipeline/sirius_pipeline_converter.cpp` → Pipeline-Construction Map (Step 4 core)

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), the
**Step-4 / pipeline-construction** reading (added after this file was found to be the gap
where Step 4 actually lives). Tagged **study** — at 1,343 lines it's the biggest file in
`src/pipeline/` and the place most of `docs/super-sirius/execution-flow.md` **Step 4**
(sub-steps 4b/4c/4d) is implemented. Read with that doc's Step 4 open and with
[`sirius_meta_pipeline.md`](sirius_meta_pipeline.md) (Step 4a) first.

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> study-note links are relative to this file. Line numbers as of 2026-06-16 (pull
> `d9172de6`) — re-grep if moved.

## Why this file matters (and why you couldn't find Step 4)

`execution-flow.md` headlines Step 4 as `sirius_engine.cpp::initialize_internal()`, but
that function is a ~50-line **orchestrator** ([`../sirius_engine.md`](../sirius_engine.md)).
The actual construction is delegated:

```
initialize_internal()                    sirius_engine.cpp:350
├─ root.build() / root.ready()           ─▶ sirius_meta_pipeline.cpp   · Step 4a
├─ converter.convert(root)               ─▶ THIS FILE                  · Step 4b + 4d
└─ materialize_repository_wiring(result) ─▶ repository_wiring_materializer.cpp · Step 4c
```

So **this file is where the operator splitting you've seen referenced all over Weeks 2–3
is born** — the local→merge aggregate pairs, the `PARTITION`/`CONCAT` around joins, the
4-phase sort. If a map ever said "the engine creates the pair…", the *real* site is here
(`construct_sirius_specific_operator`, 61, + the `split_*` methods).

## The key idea first: meta-pipelines in → execution pipelines out, via a 6-step `convert()`

The meta-pipeline tree (Step 4a) is DuckDB-shaped: source→…→sink lists with build-side
children. The converter turns that into **Sirius execution pipelines** by (a) deep-copying,
(b) **splitting** operators into Sirius-specific multi-pipeline shapes, (c) emitting wiring
descriptors, (d) finalizing dependencies. The whole thing is one public method,
`convert()` (123), calling six private phases in order:

## Call sequence

`─▶` leaves this file; `· Step N` = `execution-flow.md` step.

```
convert(root_meta_pipeline)                         :123   ◀─ sirius_engine::initialize_internal :380
├─ schedule_and_copy_pipelines(root)                :144   Phase 1: dependency-order + deep-copy
│     └─ construct_sirius_specific_operator(op)      :61    factory: generic op → Sirius op (scan/merge)
├─ split_pipelines(copied)                           :923   Phase 2: per-pipeline operator splitting ─┐
│   ├─ split_table_scan_source       :360  TABLE_SCAN → DUCKDB_SCAN / GPU_PARQUET_SCAN               │
│   │     ├─ insert_parquet_scan_operator        :237                                                 │
│   │     └─ insert_duckdb_native_scan_operator   :283                                                │  · Step 4b
│   ├─ split_cpu_source              :385                                                             │
│   ├─ split_intermediate_joins      :429  (mid-pipeline joins)                                       │
│   ├─ split_join_sink               :547  HASH_JOIN → PARTITION + CONCAT (both sides)                │
│   ├─ split_group_aggregate_sink    :624  HASH_GROUP_BY → PARTITION + MERGE_GROUP_BY                 │
│   ├─ split_order_by_sink           :686  ORDER_BY → SORT_SAMPLE → SORT_PARTITION → MERGE_SORT       │
│   ├─ split_top_n_sink              :780  TOP_N → MERGE_TOP_N                                        │
│   └─ split_delim_join_sink         :813  DELIM_JOIN → partition_join + distinct branches           │
├─ compute_repository_wiring()                       :960   Phase 3: emit repository_wiring descriptors · Step 4c
├─ setup_pipeline_parents()                          :1103  Phase 4: parent/child deps
├─ finalize_pipeline_structure()                     :1119  Phase 4: merge sink into operators vector  · Step 4d
├─ link_join_partition_siblings()                    :1134  Phase 4: link build/probe PARTITION pair
└─ configure_partition_min_partitions()              :1176  multi-GPU partition floor (num_gpus)
returns pipeline_conversion_result ─▶ materialize_repository_wiring(...)  · Step 4c (runtime half)
```

## Phase 1 — schedule + copy + the operator factory

`schedule_and_copy_pipelines` (144) walks the meta-pipelines in dependency order and
**deep-copies** them (the plan is immutable; the converter owns its mutable copy).
`construct_sirius_specific_operator` (61) is the **factory** that maps a generic operator to
its Sirius execution form — and it's exactly the local→merge sink half you read in Week 2/3:

| Generic op | → constructed Sirius op |
|---|---|
| `TABLE_SCAN` | `sirius_physical_{parquet,iceberg,duckdb}_scan` (by bind data) |
| `HASH_GROUP_BY` | `sirius_physical_grouped_aggregate_merge` |
| `ORDER_BY` | `sirius_physical_merge_sort` |
| `TOP_N` | `sirius_physical_top_n_merge` |
| `UNGROUPED_AGGREGATE` | `sirius_physical_ungrouped_aggregate_merge` |

This is a **pure factory** (no engine/context state) — note that's the missing half of the
"who creates the merge operator" question: the *local* operator is in the plan; the *merge*
sink is minted here.

## Phase 2 — splitting (the heart, Step 4b)

Each `split_*` method (dispatched from `split_pipelines`, 923) rewrites one pipeline into
the Sirius multi-pipeline shape, **inserting** the breaker operators into
`inserted_operators_` (owned by the result). This is the single source of truth for the
plan transforms `execution-flow.md` Step 4b lists, and it's *the* file to read to
understand why a join becomes three pipelines or a GROUP BY becomes a PARTITION→MERGE pair.
Cross-references:
- joins → [`../op/sirius_physical_hash_join.md`](../op/sirius_physical_hash_join.md) +
  [`../op/sirius_physical_partition.md`](../op/sirius_physical_partition.md)
- grouped agg → [`../op/sirius_physical_grouped_aggregate.md`](../op/sirius_physical_grouped_aggregate.md)
- ungrouped agg → [`../op/sirius_physical_ungrouped_aggregate.md`](../op/sirius_physical_ungrouped_aggregate.md)
- top-n → [`../op/sirius_physical_top_n.md`](../op/sirius_physical_top_n.md)

## Phase 3 — wiring descriptors (Step 4c), and the plan-time/runtime split

`compute_repository_wiring` (960) does **not** create repositories — it emits a list of
**`repository_wiring`** descriptors (a plan-time, purely topological "sink→source edge" with
a `port_id` + `MemoryBarrierType`). The runtime half lives in a separate small file:

> **Folded-in file: `repository_wiring_materializer.cpp` (65 lines).**
> `materialize_repository_wiring(wirings, data_repo_manager)` (called from
> `initialize_internal:383`, *after* `convert()` returns) turns each descriptor into a live
> `cucascade::shared_data_repository` + attaches an operator `port`. Two special cases it
> preserves: `RIGHT_DELIM_JOIN` also gets a `FULL` port on its `partition_join` sibling;
> `LEFT_DELIM_JOIN` as a destination throws. **Precondition:** pipeline ids assigned first
> (`finalize_pipeline_structure`), since ports are kept ordered by pipeline id.

This plan-time-descriptor / runtime-materialize split is the cleanest thing to take away:
the converter is **pure plan math**; nothing touches cuCascade until materialization. (Port
+ barrier *semantics* themselves are in
[`../../weeks/week4-5-concepts.md`](../../weeks/week4-5-concepts.md) Part 2 and
`docs/super-sirius/data-management.md`.)

## Phase 4 — finalize

`setup_pipeline_parents` (1103) wires parent/child pipeline dependencies;
`finalize_pipeline_structure` (1119) merges each sink into its `operators` vector and
assigns pipeline ids; `link_join_partition_siblings` (1134) connects the build/probe
`PARTITION` pair (the `_sibling_partition_op` you met in the partition map);
`configure_partition_min_partitions` (1176) applies the multi-GPU partition floor
(`num_gpus`, no-op for single-GPU). The result `pipeline_conversion_result`
(`scheduled_pipelines`, `inserted_operators`, `repository_wirings`, `meta_pipeline_count`)
is moved back to the engine as `new_scheduled`.

## Types fundamental to *this* file

- **`pipeline_conversion_result`** *(this file's hpp)* — the four outputs: the ordered
  pipelines, the owned inserted operators, the wiring descriptors, the count. **Think:**
  "everything `initialize_internal` needs to hand the scheduler."
- **`repository_wiring`** *(`repository_wiring.hpp`)* — one plan-time sink→source edge
  (`port_id`, `barrier_type`, `source_op`, source/dest pipeline). **Think:** "a wire on
  paper; `materialize_*` solders it."
- **`sirius_meta_pipeline`** *(input; [`sirius_meta_pipeline.md`](sirius_meta_pipeline.md))*
  — the DuckDB-shaped pipeline tree this converter consumes. **Think:** "Step 4a's output =
  this file's input."
- **`sirius_pipeline`** *(`sirius_pipeline.hpp`, 475 lines)* — the execution pipeline object:
  an ordered `operators` list with `source`/`sink` aliases, `get_operators()`,
  `set_pipeline_id()`. **Think:** "the runnable unit the scheduler iterates" (you saw it as
  `pipeline->get_operators()` in [`gpu_pipeline_task`](gpu_pipeline_task.md), run by the
  [executor](gpu_pipeline_executor.md)).
- **`pipeline_build_context` / `sirius_pipeline_build_state`** *(plan-time)* — config
  (`num_gpus`, `preserve_insertion_order`) + build bookkeeping threaded through 4a→4b.
  **Think:** "the scratch state for one query's construction."

## Takeaway

This file *is* Step 4's substance: `convert()` deep-copies the meta-pipelines, **splits**
operators into Sirius's multi-pipeline shapes (the PARTITION/CONCAT/MERGE/sort transforms),
emits topological wiring descriptors, and finalizes dependencies — then the engine
materializes the wiring into live repositories. With this mapped, the Step 4 trail no longer
goes cold at `initialize_internal`: 4a is [`sirius_meta_pipeline.md`](sirius_meta_pipeline.md),
4b/4d are here, 4c is the materializer above. Next, the constructed pipelines flow into the
scheduler tier ([`task_scheduler.md`](task_scheduler.md) →
[`gpu_pipeline_executor.md`](gpu_pipeline_executor.md), Steps 5/7).
