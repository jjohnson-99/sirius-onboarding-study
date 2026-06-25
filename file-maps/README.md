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

## Maps by role

| Role | Map(s) |
|---|---|
| **Doorways** | [sirius_extension](sirius_extension.md), [transparent/sirius_optimizer_extension](transparent/sirius_optimizer_extension.md) |
| **Lifecycle** | [sirius_interface](sirius_interface.md) |
| **Engine** | [sirius_engine](sirius_engine.md) |
| **Pipeline construction (Step 4)** | [pipeline/sirius_meta_pipeline](pipeline/sirius_meta_pipeline.md) (4a build), [pipeline/sirius_pipeline_converter](pipeline/sirius_pipeline_converter.md) (4b–4d split/wire) |
| **Ownership** | [sirius_context](sirius_context.md) |
| **Plan** | [planner/sirius_physical_plan_generator](planner/sirius_physical_plan_generator.md) (dispatcher), [planner/plan-builders](planner/plan-builders.md) (per-node builders + `from_duckdb` call sites), [aggregate](planner/sirius_plan_aggregate.md) + [comparison_join](planner/sirius_plan_comparison_join.md) (heavyweight builders), [get](planner/sirius_plan_get.md) (`LOGICAL_GET` scan builder + pure-filter classification) |
| **Operators** | [op/sirius_physical_operator](op/sirius_physical_operator.md) (base), [limit](op/sirius_physical_limit.md), [duckdb_scan](op/sirius_physical_duckdb_scan.md), [table_scan](op/sirius_physical_table_scan.md) (generic scan + filter/project `execute()`), [ungrouped_aggregate](op/sirius_physical_ungrouped_aggregate.md), [grouped_aggregate](op/sirius_physical_grouped_aggregate.md), [top_n](op/sirius_physical_top_n.md), [partition](op/sirius_physical_partition.md), [hash_join](op/sirius_physical_hash_join.md) |
| **Expressions** | [expression/ast](expression/ast.md) (the `sirius::ast::node` AST + `from_duckdb` — producer entry), [expression_executor/gpu_expression_executor](expression_executor/gpu_expression_executor.md) (FILTER/PROJECTION compute), [expression_executor/gpu_expression_translator](expression_executor/gpu_expression_translator.md) (mixed-join AST) |
| **Scheduler tier** | [pipeline/task_scheduler](pipeline/task_scheduler.md) (where/when), [pipeline/gpu_pipeline_executor](pipeline/gpu_pipeline_executor.md) (per-GPU dispatch), [pipeline/gpu_pipeline_task](pipeline/gpu_pipeline_task.md) (the resumable task it runs), [pipeline/sirius_pipeline](pipeline/sirius_pipeline.md) (the runnable unit + completion bookkeeping), [creator/task_creator](creator/task_creator.md) (what runs next) |
| **Scan** | [op/scan/duckdb_scan_executor](op/scan/duckdb_scan_executor.md) (host-I/O scan executor + cache), [op/scan/duckdb_native_gpu_ingestible](op/scan/duckdb_native_gpu_ingestible.md) (native decoder ingestible + `post_filter_and_project`), [op/scan/scan_plan](op/scan/scan_plan.md) (read/output layout + pure-filter classification) |
| **Memory & degradation** | [memory/sirius_memory_reservation_manager](memory/sirius_memory_reservation_manager.md) (tiers/reservations), [downgrade/downgrade_executor](downgrade/downgrade_executor.md) (spilling), [fallback](fallback.md) (CPU fallback) |

Full one-line descriptions: the [top-level README](README.md). Which maps carry a call
sequence (orchestration files only) is per the rule in the methodology notes.
