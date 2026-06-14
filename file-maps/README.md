# File Maps

One map per Sirius source file (mirroring `src/`). Each annotates the file's functions and
key types against `docs/super-sirius/execution-flow.md`; the **orchestration** maps also
carry a `## Call sequence` tree. This page stitches those trees into one **end-to-end call
walk** — the control-flow spine from "query enters" to "pipelines launched."

(Structural parallel to [`reference/explainers/README.md`](reference/explainers/README.md),
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
        ▼   engine hands new_scheduled to the scheduler
SCHEDULER   task_scheduler / task_creator / gpu_pipeline_executor [Week 4 — maps to come]
```

Following the arrows and `· Step N` tags walks Steps **0 → 1 → 3 → 4 → 5 → 9** in order.
This is just the connective overview — the per-function detail lives in each map's own
`## Call sequence`:

- [`file-maps/sirius_extension.md`](file-maps/sirius_extension.md#call-sequence) — load + explicit doorway
- [`file-maps/transparent/sirius_optimizer_extension.md`](file-maps/transparent/sirius_optimizer_extension.md#call-sequence) — transparent doorway
- [`file-maps/sirius_interface.md`](file-maps/sirius_interface.md#call-sequence) — lifecycle (Steps 3 & 9)
- [`file-maps/sirius_engine.md`](file-maps/sirius_engine.md#call-sequence) — build + launch (Steps 4 & 5)

For the *conceptual* version of the same trace (operators, scan, result included), see
[`weeks/week2-concepts.md`](weeks/week2-concepts.md).

## Maps by role

| Role | Map(s) |
|---|---|
| **Doorways** | [sirius_extension](file-maps/sirius_extension.md), [transparent/sirius_optimizer_extension](file-maps/transparent/sirius_optimizer_extension.md) |
| **Lifecycle** | [sirius_interface](file-maps/sirius_interface.md) |
| **Engine** | [sirius_engine](file-maps/sirius_engine.md) |
| **Ownership** | [sirius_context](file-maps/sirius_context.md) |
| **Plan** | [planner/sirius_physical_plan_generator](file-maps/planner/sirius_physical_plan_generator.md) |
| **Operators** | [op/sirius_physical_operator](file-maps/op/sirius_physical_operator.md) (base), [limit](file-maps/op/sirius_physical_limit.md), [duckdb_scan](file-maps/op/sirius_physical_duckdb_scan.md), [ungrouped_aggregate](file-maps/op/sirius_physical_ungrouped_aggregate.md), [grouped_aggregate](file-maps/op/sirius_physical_grouped_aggregate.md) |

Full one-line descriptions: the [top-level README](README.md). Which maps carry a call
sequence (orchestration files only) is per the rule in the methodology notes.
