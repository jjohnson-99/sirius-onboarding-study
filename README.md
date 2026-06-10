# Sirius Onboarding Study Notes

Working notes from an onboarding study session on the Sirius codebase — a plan plus
supporting explainers, for a contributor who knows modern C++ but not databases.
Generated collaboratively; meant for quick navigation of that discussion.

## Layout

```
sirius-onboarding-study/
├── README.md                     this index
├── onboarding-path.md            the plan (start here)
├── onboarding-coverage-estimate.md
├── weeks/                        one concepts explainer per onboarding week
│   ├── week1-concepts.md
│   └── week2-concepts.md
├── reference/                    shared references the per-file maps link to
│   └── duckdb-types-glossary.md
└── file-maps/                    one map per source file, MIRRORING sirius/src/
    ├── sirius_extension.md        ↔ src/sirius_extension.cpp
    ├── sirius_interface.md        ↔ src/sirius_interface.cpp
    ├── sirius_engine.md           ↔ src/sirius_engine.cpp
    ├── sirius_context.md          ↔ src/sirius_context.{hpp,cpp}
    ├── planner/
    │   └── sirius_physical_plan_generator.md  ↔ src/planner/...plan_generator.cpp
    └── op/
        ├── sirius_physical_operator.md            ↔ src/op/sirius_physical_operator.cpp
        ├── sirius_physical_limit.md               ↔ src/op/sirius_physical_limit.cpp
        ├── sirius_physical_duckdb_scan.md         ↔ src/op/sirius_physical_duckdb_scan.cpp
        └── sirius_physical_ungrouped_aggregate.md ↔ src/op/sirius_physical_ungrouped_aggregate.cpp
```

`file-maps/` mirrors the repo's `src/` tree: a source file at `src/<path>/<name>.cpp`
gets its map at `file-maps/<path>/<name>.md` (e.g. `src/op/sirius_physical_limit.cpp`
→ `file-maps/op/sirius_physical_limit.md`). We add `weeks/` and `file-maps/` entries
as we work through [`onboarding-path.md`](onboarding-path.md); not all exist yet.

## Index

| File | What it is |
|------|------------|
| [onboarding-path.md](onboarding-path.md) | The ~6–7 week onboarding plan: weekly phases, checkable tasks, effort tags, milestones, and a DB-theory resource list. **Start here.** |
| [onboarding-coverage-estimate.md](onboarding-coverage-estimate.md) | How much of the codebase the plan actually has you read — tiered line/percentage estimate (the "spine + deep dives" rationale). |
| [weeks/week1-concepts.md](weeks/week1-concepts.md) | Week 1 checkpoint explainer: physical plan, operator, pipeline, hash join, and how SQL becomes GPU work — cross-referenced to the Week 1 readings. |
| [weeks/week2-concepts.md](weeks/week2-concepts.md) | Week 2 synthesis: the target query traced end-to-end through 8 stages, tying together every Week 2 file-map and mapped onto `execution-flow.md`. |
| [reference/duckdb-types-glossary.md](reference/duckdb-types-glossary.md) | Shared reference for the recurring DuckDB types (`ClientContext`, `DataChunk`, `LogicalOperator`, …). Per-file maps link here instead of re-explaining. |
| [file-maps/sirius_extension.md](file-maps/sirius_extension.md) | `src/sirius_extension.cpp` — the doorway: extension load (Step 0) + the explicit `gpu_execution` path (Step 1b). |
| [file-maps/sirius_interface.md](file-maps/sirius_interface.md) | `src/sirius_interface.cpp` (Steps 3 & 9): query-lifecycle call graph, the `sirius_active_query` state machine, the DuckDB methods it forks. |
| [file-maps/sirius_engine.md](file-maps/sirius_engine.md) | `src/sirius_engine.cpp` (Steps 4 & 5): build pipelines + launch into the scheduler; the orchestration core. *read closely* |
| [file-maps/sirius_context.md](file-maps/sirius_context.md) | `src/sirius_context.hpp` (Day 3): the ownership hierarchy + transparent-execution interceptor; the `"sirius_state"` target. *read closely* |
| [file-maps/planner/sirius_physical_plan_generator.md](file-maps/planner/sirius_physical_plan_generator.md) | `src/planner/sirius_physical_plan_generator.cpp`: logical→physical translation switch; Sirius's supported-feature boundary. |
| [file-maps/op/sirius_physical_operator.md](file-maps/op/sirius_physical_operator.md) | `src/op/sirius_physical_operator.cpp`: the base operator interface (execute/sink/source/build_pipelines/task-hint) every operator implements. |
| [file-maps/op/sirius_physical_limit.md](file-maps/op/sirius_physical_limit.md) | `src/op/sirius_physical_limit.cpp` (warm-up): the simplest `execute()` — parallel LIMIT via atomic claim + `cudf::slice`. |
| [file-maps/op/sirius_physical_duckdb_scan.md](file-maps/op/sirius_physical_duckdb_scan.md) | `src/op/sirius_physical_duckdb_scan.cpp` (tiny): the `DUCKDB_SCAN` source — a scan *descriptor*; execution lives in the scan executor. |
| [file-maps/op/sirius_physical_ungrouped_aggregate.md](file-maps/op/sirius_physical_ungrouped_aggregate.md) | `src/op/sirius_physical_ungrouped_aggregate.cpp`: the two-phase local→merge aggregate (`cudf::reduce`); closes the Week 2 trace. |

## Notes

- These notes live **outside** the Sirius repo (one level above it, beside the
  `sirius/` checkout) so they are never tracked or committed.
- These are study aids, not project documentation. Authoritative docs live in
  [`../sirius/docs/super-sirius/`](../sirius/docs/super-sirius/).
- Source/doc paths inside these notes are relative to the **Sirius repo root** at
  `../sirius/` (e.g. `src/op/...` means `../sirius/src/op/...`).
- Line numbers cited in the `file-maps/` notes were accurate as of 2026-06-10;
  re-confirm by grepping the function name if the source has moved.
