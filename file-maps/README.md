# File Maps

One map per Sirius source file (mirroring `src/`). Each annotates the file's functions and
key types against `docs/super-sirius/execution-flow.md`; the **orchestration** maps also
carry a `## Call sequence` tree. This page stitches those trees into one **end-to-end call
walk** — the control-flow spine from "query enters" to "pipelines launched."

(Structural parallel to [`reference/explainers/README.md`](../reference/explainers/README.md),
which holds the *conceptual* reading thread; this is the *code-level* spine.)

## The orchestration spine

Plain SQL or `CALL gpu_execution(...)` enters through one of **two doorways**; both converge
on `sirius_interface::sirius_execute_query`, which drives `sirius_engine`, which launches the
`task_scheduler`:

```
ENTRY ── two doorways, both ending at sirius_execute_query
  ├ explicit     CALL gpu_execution('…')
  │              GPUExecutionBind → GPUExecutionFunction          [sirius_extension · Step 0 / 1b]
  └ transparent  plain SQL → optimizer hooks → OnFinalizePrepare
                 → PhysicalSiriusExecution::GetDataInternal       [transparent  · Step 1 / 2]
        │
        ▼   both call  iface->sirius_execute_query(…)
LIFECYCLE   sirius_execute_query → pending (build) · execute (run) · fetch
                                                                  [sirius_interface · Step 3, 9]
        │
        ▼   pending → engine.initialize ;  execute → engine.execute
ENGINE      initialize → build pipelines ;  execute → launch + block
                                                                  [sirius_engine · Step 4, 5]
        │
        ▼   initialize_internal delegates Step 4 to src/pipeline/
CONSTRUCT   meta_pipeline (4a: group by sink) → converter (4b–4d: split + wire)
                                                                  [pipeline · Step 4]
        │
        ▼   engine hands new_scheduled to the scheduler
SCHEDULER   task_creator (what runs) · task_scheduler (where/when) · gpu_pipeline_executor (run)
                                                                  [Week 4 — see maps below]
        │
        ▼   sources feed the scheduler; the join is the capstone consumer
SOURCES/OPS duckdb_scan_executor (scan in) · sirius_physical_hash_join (3-mode join)
                                                                  [Week 5]
```

**Scheduler-tier roles** (Week 4): three components, three jobs —
[`task_creator`](creator/task_creator.md) decides *what* runs next (hint-chain recursion);
[`task_scheduler`](pipeline/task_scheduler.md) decides *where/when* (pull-signal matcher,
device preference); [`gpu_pipeline_executor`](pipeline/gpu_pipeline_executor.md) (one per
GPU) *dispatches* it (reserve→dispatch, OOM retry); and the
[`gpu_pipeline_task`](pipeline/gpu_pipeline_task.md) itself is what *runs* — the
per-task-device contract, the operator loop, and resumable OOM. The host-I/O
[`duckdb_scan_executor`](op/scan/duckdb_scan_executor.md) is the source-side sibling.

Following the arrows and `· Step N` tags walks Steps **0 → 1 → 3 → 4 → 5 → 9** in order.
This is just the connective overview — the per-function detail lives in each map's own
`## Call sequence`:

- [`file-maps/sirius_extension.md`](sirius_extension.md#call-sequence) — load + explicit doorway
- [`file-maps/transparent/sirius_optimizer_extension.md`](transparent/sirius_optimizer_extension.md#call-sequence) — transparent doorway
- [`file-maps/sirius_interface.md`](sirius_interface.md#call-sequence) — lifecycle (Steps 3 & 9)
- [`file-maps/sirius_engine.md`](sirius_engine.md#call-sequence) — build + launch (Steps 4 & 5)
- [`file-maps/pipeline/sirius_meta_pipeline.md`](pipeline/sirius_meta_pipeline.md) — Step 4a (build pipelines)
- [`file-maps/pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md#call-sequence) — Step 4b–4d (split + wire)

For the *conceptual* version of the same trace (operators, scan, result included), see
[`weeks/week2-concepts.md`](../weeks/week2-concepts.md).

## The scan spine

The orchestration spine above stops at "sources feed the scheduler." This is the spine *inside*
the scan source — the code-level companion to the
[`scan-pipeline-and-filter-project`](../notes/walkthroughs/scan-pipeline-and-filter-project.md)
walkthrough. The one framing to hold onto: **post-`#871` every `TABLE_SCAN` becomes a single
unified `GPU_SCAN`, and that GPU path is a producer/consumer pair that never call each other —
they meet at a queue.**

```
TABLE_SCAN                                           [planner/sirius_plan_get — builds config,
   │  split_table_scan_source (converter:360)         classifies pure-filter columns]
   │  rewrites EVERY scan into GPU_SCAN:
   ├─ parquet_scan / read_parquet → GPU_SCAN + parquet_gpu_ingestible
   └─ seq_scan                    → GPU_SCAN + duckdb_native_gpu_ingestible
        (any other function → throws "Unsupported scan function")

   ✗ DUCKDB_SCAN / ICEBERG_SCAN / GPU_PARQUET_SCAN  — pre-#871 refined operator subtypes;
     no longer produced (see "vestigial" note below). The super-sirius/scan.md mermaid still
     shows these because that doc predates #871.
```

### GPU family (`GPU_SCAN`) — producer ──▶ queue ──▶ consumer

The two halves run on different threads and are wired together by the scan_manager. They share
**one ingestible** (the producer emits splits *from* it; the consumer decodes splits *through* it)
and communicate **only** through the `split_connector`:

```
PRODUCER  scan_manager::prepare_for_query                       [scan_manager/sirius_scan_manager]
   factory builds the ingestible · installs it on the op · gives the op a fresh split_connector
   fire-and-forget split_provider ──drives──▶ ingestible.next_split_provider / run_batch
                                                     │ pushes splits (row-group ranges + metadata)
                                                     ▼
   ════════════════════════════════════  split_connector  ◀── the meeting point ══════════════
                                                     │ operator pulls (get_next_task_input_data)
                                                     ▼
CONSUMER  sirius_gpu_scan_operator::execute(split)             [op/scan/sirius_gpu_scan_operator]
   materialize_table  ──▶  (skip if reader already did it)  post_filter_and_project  ──▶ data_batch
```

The **ingestible is the polymorphic seam** — same `io::gpu_ingestible` interface, three
implementations the [factory](scan_manager/sirius_scan_manager.md) picks between, and **where the
filter/project actually runs differs per implementation** (this is the crux the FILTER→PROJECTION
gather ticket turns on):

| Ingestible | Chosen when | Filter / project happens in |
|---|---|---|
| [`parquet_gpu_ingestible`](op/scan/parquet_gpu_ingestible.md) | fresh parquet read | **`materialize_table`** (reader pushdown, else a post-decode `select` gather); `post_filter_and_project` = assembly only |
| [`duckdb_native_gpu_ingestible`](op/scan/duckdb_native_gpu_ingestible.md) | fresh base-table read | **`post_filter_and_project`** — this is the gather (the ticket's edit target) |
| [`pinned_table_gpu_ingestible`](op/scan/pinned_table_gpu_ingestible.md) | pinned-cache hit (`pin_table`) | **`post_filter_and_project`** (filter `select` + assembly); **skips `materialize_table`** |

All three read the same [`scan_plan`](op/scan/scan_plan.md) — the keep-vs-pure-filter layout the
planner's `projection_ids` becomes. That plan, plus the `filter_state` an ingestible returns from
`materialize_table`, is what lets the consumer **skip** `post_filter_and_project` when the reader
already filtered+projected (`ROW_FILTERED_AND_PROJECTED`).

### CPU family (`DUCKDB_SCAN`) — ⚠️ vestigial post-`#871`

```
sirius_physical_duckdb_scan (source)                          [op/sirius_physical_duckdb_scan]
   duckdb_scan_executor (host thread pool) schedules          [op/scan/duckdb_scan_executor]
      duckdb_scan_task: pump DuckDB table function → pinned host batch → data_repo
                                                              [op/scan/duckdb_scan_task]
```

This *was* the CPU-staging path: no scan_manager, no ingestible, no split_connector; filters
withheld from DuckDB (`nullptr`) and applied downstream on GPU. **It is no longer reachable** —
`sirius_physical_duckdb_scan` is built only by `construct_sirius_specific_operator`
(`sirius_engine.cpp:231`, zero callers; converter free-fn `:85`, called only with agg/distinct
merge ops, never `TABLE_SCAN`), and `split_table_scan_source` routes `seq_scan` to the **GPU**
family instead. Don't conflate `GPU_SCAN` (live) with `DUCKDB_SCAN` (vestigial) — and note the
near-identical name `duckdb_native_gpu_ingestible` belongs to the *GPU* family, not here. Kept in
the maps as the foil for that naming trap, not as a live alternative. See
[`duckdb_scan_task`](op/scan/duckdb_scan_task.md) for the full vestigial note.

**Reading order** (mirrors data flow): the planner ([`sirius_plan_get`](planner/sirius_plan_get.md))
→ the layout ([`scan_plan`](op/scan/scan_plan.md)) → producer
([`sirius_scan_manager`](scan_manager/sirius_scan_manager.md)) → consumer
([`sirius_gpu_scan_operator`](op/scan/sirius_gpu_scan_operator.md)) → the three ingestibles. The
walkthrough's mermaid diagram shows the same flow with the runtime (not reading) ordering.

## Maps by role

| Role | Map(s) |
|---|---|
| **Doorways** | [sirius_extension](sirius_extension.md), [transparent/sirius_optimizer_extension](transparent/sirius_optimizer_extension.md) |
| **Lifecycle** | [sirius_interface](sirius_interface.md) |
| **Engine** | [sirius_engine](sirius_engine.md) |
| **Pipeline construction (Step 4)** | [pipeline/sirius_meta_pipeline](pipeline/sirius_meta_pipeline.md) (4a build), [pipeline/sirius_pipeline_converter](pipeline/sirius_pipeline_converter.md) (4b–4d split/wire) |
| **Ownership** | [sirius_context](sirius_context.md) |
| **Plan** | [planner/sirius_physical_plan_generator](planner/sirius_physical_plan_generator.md) (dispatcher), [planner/plan-builders](planner/plan-builders.md) (per-node builders + `from_duckdb` call sites), [aggregate](planner/sirius_plan_aggregate.md) + [comparison_join](planner/sirius_plan_comparison_join.md) (heavyweight builders), [get](planner/sirius_plan_get.md) (`LOGICAL_GET` scan builder + pure-filter classification) |
| **Operators** | [op/sirius_physical_operator](op/sirius_physical_operator.md) (base), [limit](op/sirius_physical_limit.md), [duckdb_scan](op/sirius_physical_duckdb_scan.md), [table_scan](op/sirius_physical_table_scan.md) (generic scan; config carrier + bypassed filter/project copy), [ungrouped_aggregate](op/sirius_physical_ungrouped_aggregate.md), [grouped_aggregate](op/sirius_physical_grouped_aggregate.md), [top_n](op/sirius_physical_top_n.md), [partition](op/sirius_physical_partition.md), [hash_join](op/sirius_physical_hash_join.md) |
| **Expressions** | [expression/ast](expression/ast.md) (the `sirius::ast::node` AST + `from_duckdb` — producer entry), [expression_executor/gpu_expression_executor](expression_executor/gpu_expression_executor.md) (FILTER/PROJECTION compute), [expression_executor/gpu_expression_translator](expression_executor/gpu_expression_translator.md) (mixed-join AST) |
| **Scheduler tier** | [pipeline/task_scheduler](pipeline/task_scheduler.md) (where/when), [pipeline/gpu_pipeline_executor](pipeline/gpu_pipeline_executor.md) (per-GPU dispatch), [pipeline/gpu_pipeline_task](pipeline/gpu_pipeline_task.md) (the resumable task it runs), [pipeline/sirius_pipeline](pipeline/sirius_pipeline.md) (the runnable unit + completion bookkeeping), [creator/task_creator](creator/task_creator.md) (what runs next) |
| **Scan** | [op/scan/sirius_gpu_scan_operator](op/scan/sirius_gpu_scan_operator.md) (unified `GPU_SCAN` source: pull splits → drive ingestible), [op/scan/duckdb_scan_executor](op/scan/duckdb_scan_executor.md) (host-I/O scan executor + cache), [op/scan/duckdb_scan_task](op/scan/duckdb_scan_task.md) (CPU-staging scan task: DuckDB table function → pinned host batch; **vestigial post-`#871`**), [op/scan/duckdb_native_gpu_ingestible](op/scan/duckdb_native_gpu_ingestible.md) (native decoder ingestible + `post_filter_and_project`), [op/scan/parquet_gpu_ingestible](op/scan/parquet_gpu_ingestible.md) (parquet fresh-read ingestible; filter in `materialize_table`, assembly-only post), [op/scan/pinned_table_gpu_ingestible](op/scan/pinned_table_gpu_ingestible.md) (pinned-cache ingestible; GPU/HOST tier, skips `materialize_table`), [op/scan/scan_plan](op/scan/scan_plan.md) (read/output layout + pure-filter classification) |
| **Scan-manager** | [scan_manager/sirius_scan_manager](scan_manager/sirius_scan_manager.md) (split-production engine: per-query ingestible build + fire-and-forget split workers; owns IO backends) |
| **Memory & degradation** | [memory/sirius_memory_reservation_manager](memory/sirius_memory_reservation_manager.md) (tiers/reservations), [downgrade/downgrade_executor](downgrade/downgrade_executor.md) (spilling), [fallback](fallback.md) (CPU fallback) |

Full one-line descriptions: the [top-level README](README.md). Which maps carry a call
sequence (orchestration files only) is per the rule in the methodology notes.
