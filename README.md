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
│   ├── week2-concepts.md
│   └── week3-concepts.md
├── reference/                    shared references the per-file maps link to
│   ├── duckdb-types-glossary.md
│   └── explainers/               long-form background concepts (one topic per file)
│       ├── oltp-vs-olap.md
│       ├── columnar-vs-row-storage.md
│       ├── vectorized-execution.md
│       ├── gpu-vs-cpu-for-databases.md
│       ├── gpu-warps-and-execution.md
│       ├── cuda-streams-and-async.md
│       ├── gpu-memory-and-spilling.md
│       ├── cudf.md
│       ├── cucascade.md
│       ├── rmm.md
│       ├── rapids-gpu-data-ecosystem.md
│       ├── duckdb-extension-api.md
│       ├── substrait-and-plan-irs.md
│       ├── filters-and-expression-evaluation.md
│       ├── types-duckdb-cudf-sirius.md
│       ├── nulls-and-validity.md
│       ├── testing-sirius.md
│       ├── push-vs-pull-execution.md
│       ├── duckdb-table-functions.md
│       ├── client-connections.md
│       ├── aggregation-and-group-by.md
│       ├── morsel-driven-parallelism.md
│       ├── hash-join-build-probe.md
│       ├── sorting-and-order-by.md
│       ├── parquet-format.md
│       ├── iceberg-table-format.md
│       └── mvcc-concurrency-control.md
└── file-maps/                    one map per source file, MIRRORING sirius/src/
    ├── README.md                  index + the orchestration-spine call walk
    ├── sirius_extension.md        ↔ src/sirius_extension.cpp
    ├── sirius_interface.md        ↔ src/sirius_interface.cpp
    ├── sirius_engine.md           ↔ src/sirius_engine.cpp
    ├── sirius_context.md          ↔ src/sirius_context.{hpp,cpp}
    ├── transparent/
    │   └── sirius_optimizer_extension.md  ↔ src/transparent/{sirius_optimizer_extension,physical_sirius_execution}.cpp
    ├── planner/
    │   └── sirius_physical_plan_generator.md  ↔ src/planner/...plan_generator.cpp
    ├── expression_executor/
    │   ├── gpu_expression_executor.md     ↔ src/expression_executor/gpu_expression_executor.cpp (+ specializations/)
    │   └── gpu_expression_translator.md   ↔ src/expression_executor/gpu_expression_translator.cpp
    └── op/
        ├── sirius_physical_operator.md            ↔ src/op/sirius_physical_operator.cpp
        ├── sirius_physical_limit.md               ↔ src/op/sirius_physical_limit.cpp
        ├── sirius_physical_duckdb_scan.md         ↔ src/op/sirius_physical_duckdb_scan.cpp
        ├── sirius_physical_ungrouped_aggregate.md ↔ src/op/sirius_physical_ungrouped_aggregate.cpp
        ├── sirius_physical_grouped_aggregate.md   ↔ src/op/sirius_physical_grouped_aggregate.cpp
        ├── sirius_physical_top_n.md               ↔ src/op/sirius_physical_top_n.cpp (+ _merge)
        └── sirius_physical_partition.md           ↔ src/op/sirius_physical_partition.cpp
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
| [weeks/week3-concepts.md](weeks/week3-concepts.md) | Week 3 synthesis: turning *downward* from the spine into the operator + expression layer — the "tree of AST trees" expression executor, a medium operator (TOP-N / PARTITION), and the first-PR workflow. |
| [reference/duckdb-types-glossary.md](reference/duckdb-types-glossary.md) | Shared reference for the recurring DuckDB types (`ClientContext`, `DataChunk`, `LogicalOperator`, …). Per-file maps link here instead of re-explaining. |
| [reference/duckdb-source-map.md](reference/duckdb-source-map.md) | Pointers into **DuckDB's own source** (not in this repo) for the internals our notes reference — query lifecycle, extension API, logical plans, core types, optimizer. Read at Sirius's pinned commit. |
| [reference/explainers/](reference/explainers/README.md) | Long-form background concepts intuitive to DB folks but not assumed of the reader — OLTP vs. OLAP, columnar vs. row storage, vectorized execution, GPU vs. CPU (+ warps/execution model, CUDA streams/async, memory & spilling), cuDF, cuCascade, RMM, RAPIDS ecosystem, filters & expression evaluation, types (DuckDB↔cuDF↔Sirius), NULLs & validity, testing, push vs. pull, DuckDB extension API, Substrait & plan IRs, table functions, client connections & config scope, aggregation & GROUP BY, morsel-driven parallelism, hash join build/probe, sorting & ORDER BY, Parquet, Iceberg, MVCC. One topic per file, each tagged with the onboarding week to **prime around** — and arranged into a "why GPU-native SQL" reading thread (see its README). |
| [file-maps/](file-maps/README.md) | **Index of the per-file maps + the orchestration-spine call walk** stitching the four doorway/lifecycle/engine maps into one end-to-end control-flow trace. |
| [file-maps/sirius_extension.md](file-maps/sirius_extension.md) | `src/sirius_extension.cpp` — the doorway: extension load (Step 0) + the explicit `gpu_execution` path (Step 1b). |
| [file-maps/transparent/sirius_optimizer_extension.md](file-maps/transparent/sirius_optimizer_extension.md) | The **primary** doorway (Steps 1–2): plain-SQL interception via optimizer hooks + `PhysicalSiriusExecution`. Covers both `src/transparent/*.cpp`; converges with the explicit path on `sirius_interface`. |
| [file-maps/sirius_interface.md](file-maps/sirius_interface.md) | `src/sirius_interface.cpp` (Steps 3 & 9): query-lifecycle call graph, the `sirius_active_query` state machine, the DuckDB methods it forks. **Where both doorways converge.** |
| [file-maps/sirius_engine.md](file-maps/sirius_engine.md) | `src/sirius_engine.cpp` (Steps 4 & 5): build pipelines + launch into the scheduler; the orchestration core. *read closely* |
| [file-maps/sirius_context.md](file-maps/sirius_context.md) | `src/sirius_context.hpp` (Day 3): the ownership hierarchy + transparent-execution interceptor; the `"sirius_state"` target. *read closely* |
| [file-maps/planner/sirius_physical_plan_generator.md](file-maps/planner/sirius_physical_plan_generator.md) | `src/planner/sirius_physical_plan_generator.cpp`: logical→physical translation switch; Sirius's supported-feature boundary. |
| [file-maps/op/sirius_physical_operator.md](file-maps/op/sirius_physical_operator.md) | `src/op/sirius_physical_operator.cpp`: the base operator interface (execute/sink/source/build_pipelines/task-hint) every operator implements. |
| [file-maps/op/sirius_physical_limit.md](file-maps/op/sirius_physical_limit.md) | `src/op/sirius_physical_limit.cpp` (warm-up): the simplest `execute()` — parallel LIMIT via atomic claim + `cudf::slice`. |
| [file-maps/op/sirius_physical_duckdb_scan.md](file-maps/op/sirius_physical_duckdb_scan.md) | `src/op/sirius_physical_duckdb_scan.cpp` (tiny): the `DUCKDB_SCAN` source — a scan *descriptor*; execution lives in the scan executor. |
| [file-maps/op/sirius_physical_ungrouped_aggregate.md](file-maps/op/sirius_physical_ungrouped_aggregate.md) | `src/op/sirius_physical_ungrouped_aggregate.cpp`: the two-phase local→merge aggregate (`cudf::reduce`); closes the Week 2 trace. |
| [file-maps/op/sirius_physical_grouped_aggregate.md](file-maps/op/sirius_physical_grouped_aggregate.md) | `src/op/sirius_physical_grouped_aggregate.cpp` (+ merge): the actual `GROUP BY` operator (`cudf::groupby`) — where the Week 2 query's grouping happens; `HASH_GROUP_BY → PARTITION → MERGE_GROUP_BY`. |
| [file-maps/expression_executor/gpu_expression_executor.md](file-maps/expression_executor/gpu_expression_executor.md) | `src/expression_executor/gpu_expression_executor.cpp` (+ `specializations/`): how FILTER/PROJECTION evaluate expressions on the GPU — the "tree of AST trees," strategies, `min_ast_size`, and the in-progress Sirius-AST migration. |
| [file-maps/expression_executor/gpu_expression_translator.md](file-maps/expression_executor/gpu_expression_translator.md) | `src/expression_executor/gpu_expression_translator.cpp`: the *separate* DuckDB→cuDF AST translator (all-or-nothing, for mixed/inequality joins) — evaluate-here vs. translate-and-hand-off. |
| [file-maps/op/sirius_physical_top_n.md](file-maps/op/sirius_physical_top_n.md) | `src/op/sirius_physical_top_n.cpp` (+ `_merge`): the Day-3 medium operator — two-phase local→merge TOP-N, and per-case cuDF primitive selection (`top_k_order` vs. full-sort fallback). |
| [file-maps/op/sirius_physical_partition.md](file-maps/op/sirius_physical_partition.md) | `src/op/sirius_physical_partition.cpp`: the harder Day-3 alternative — hash fan-out feeding joins/group-bys; derives keys from the parent op, sizes `N`, coordinates a build/probe sibling pair. A Week 4–5 on-ramp. |

## Notes

- These notes live **outside** the Sirius repo (one level above it, beside the
  `sirius/` checkout) so they are never tracked or committed.
- These are study aids, not project documentation. Authoritative docs live in the
  Sirius repo at `docs/super-sirius/`.
- **Path conventions:** links between these study notes are **relative to the file they're
  in** (normal Markdown relative paths) — a bare filename for same-directory
  (`sirius_interface.md`), `../` to go up (`../weeks/week2-concepts.md` from a `file-maps/`
  file). References into the Sirius source repo use that repo's path (e.g. `src/op/...`,
  `docs/super-sirius/...`) and won't resolve in this standalone copy.
- Line numbers cited in the `file-maps/` notes were accurate as of 2026-06-10;
  re-confirm by grepping the function name if the source has moved.
