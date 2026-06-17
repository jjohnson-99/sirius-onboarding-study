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
│   ├── week3-concepts.md
│   ├── week4-5-concepts.md
│   └── week6-7-concepts.md
├── reference/                    shared references the per-file maps link to
│   ├── duckdb-types-glossary.md
│   ├── duckdb-source-map.md
│   ├── running-sirius-on-a-cloud-gpu.md   how-to: actually run it (Linux + NVIDIA)
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
    ├── fallback.md                ↔ CPU-fallback mechanism (live path is in sirius_extension.cpp; plan's src/fallback.cpp is legacy)
    ├── transparent/
    │   └── sirius_optimizer_extension.md  ↔ src/transparent/{sirius_optimizer_extension,physical_sirius_execution}.cpp
    ├── planner/
    │   ├── sirius_physical_plan_generator.md  ↔ src/planner/...plan_generator.cpp (dispatcher)
    │   ├── plan-builders.md                   ↔ src/planner/sirius_plan_*.cpp (per-node builders family)
    │   ├── sirius_plan_aggregate.md           ↔ src/planner/sirius_plan_aggregate.cpp (heavyweight)
    │   └── sirius_plan_comparison_join.md     ↔ src/planner/sirius_plan_comparison_join.cpp (heavyweight)
    ├── expression/
    │   └── ast.md                         ↔ src/expression/ (sirius::ast::node + from_duckdb/to_duckdb — the producer entry)
    ├── expression_executor/
    │   ├── gpu_expression_executor.md     ↔ src/expression_executor/gpu_expression_executor.cpp (+ specializations/)
    │   └── gpu_expression_translator.md   ↔ src/expression_executor/gpu_expression_translator.cpp
    ├── pipeline/
    │   ├── sirius_pipeline.md             ↔ src/pipeline/sirius_pipeline.cpp (the runnable unit + plan-time builder)
    │   ├── sirius_meta_pipeline.md        ↔ src/pipeline/sirius_meta_pipeline.cpp (Step 4a build)
    │   ├── sirius_pipeline_converter.md   ↔ src/pipeline/sirius_pipeline_converter.cpp (Step 4b–4d split/wire; + repository_wiring_materializer)
    │   ├── task_scheduler.md              ↔ src/pipeline/task_scheduler.cpp (concurrency core)
    │   ├── gpu_pipeline_executor.md       ↔ src/pipeline/gpu_pipeline_executor.cpp (per-GPU executor)
    │   └── gpu_pipeline_task.md           ↔ src/pipeline/gpu_pipeline_task.cpp (the resumable task it runs)
    ├── creator/
    │   └── task_creator.md                ↔ src/creator/task_creator.cpp (what runs next)
    ├── memory/
    │   └── sirius_memory_reservation_manager.md   ↔ src/memory/sirius_memory_reservation_manager.cpp (+ defragmenter)
    ├── downgrade/
    │   └── downgrade_executor.md          ↔ src/downgrade/downgrade_executor.cpp (spilling)
    └── op/
        ├── sirius_physical_operator.md            ↔ src/op/sirius_physical_operator.cpp
        ├── sirius_physical_limit.md               ↔ src/op/sirius_physical_limit.cpp
        ├── sirius_physical_duckdb_scan.md         ↔ src/op/sirius_physical_duckdb_scan.cpp
        ├── sirius_physical_ungrouped_aggregate.md ↔ src/op/sirius_physical_ungrouped_aggregate.cpp
        ├── sirius_physical_grouped_aggregate.md   ↔ src/op/sirius_physical_grouped_aggregate.cpp
        ├── sirius_physical_top_n.md               ↔ src/op/sirius_physical_top_n.cpp (+ _merge)
        ├── sirius_physical_partition.md           ↔ src/op/sirius_physical_partition.cpp
        ├── sirius_physical_hash_join.md           ↔ src/op/sirius_physical_hash_join.cpp (the hard one)
        └── scan/
            └── duckdb_scan_executor.md            ↔ src/op/scan/duckdb_scan_executor.cpp (scan subsystem)
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
| [weeks/week4-5-concepts.md](weeks/week4-5-concepts.md) | Weeks 4–5 synthesis: the concurrency core traced through a *join* query — the three scheduler roles (creator/scheduler/executor), cross-pipeline data flow (batches/repos/ports/barriers), the scan source, and the three-mode hash join. |
| [weeks/week6-7-concepts.md](weeks/week6-7-concepts.md) | Weeks 6–7 synthesis: memory & graceful degradation traced through "what if a task needs more GPU than is free?" — tiers/reservations, defrag→spill→OOM-retry, CPU fallback (with the path correction), multi-GPU skim, and the Week-7 contribution workflow. |
| [reference/duckdb-types-glossary.md](reference/duckdb-types-glossary.md) | Shared reference for the recurring DuckDB types (`ClientContext`, `DataChunk`, `LogicalOperator`, …). Per-file maps link here instead of re-explaining. |
| [reference/duckdb-source-map.md](reference/duckdb-source-map.md) | Pointers into **DuckDB's own source** (not in this repo) for the internals our notes reference — query lifecycle, extension API, logical plans, core types, optimizer. Read at Sirius's pinned commit. |
| [reference/running-sirius-on-a-cloud-gpu.md](reference/running-sirius-on-a-cloud-gpu.md) | How-to for actually **running** Sirius (you can't on a Mac): the three hard blockers from `pixi.toml` (Linux-only, NVIDIA-CUDA-only, cc ≥ 7.5), what to rent (**aim for 2 GPUs** so the Week 4–7 scheduler/multi-device paths are live; T4/L4 class), the clone→submodules→`pixi run make`→load-extension steps, CUDA 12-vs-13 = which pixi env, cost/teardown, and what you can still do locally. |
| [reference/explainers/](reference/explainers/README.md) | Long-form background concepts intuitive to DB folks but not assumed of the reader — OLTP vs. OLAP, columnar vs. row storage, vectorized execution, GPU vs. CPU (+ warps/execution model, CUDA streams/async, memory & spilling), cuDF, cuCascade, RMM, RAPIDS ecosystem, filters & expression evaluation, types (DuckDB↔cuDF↔Sirius), NULLs & validity, testing, push vs. pull, DuckDB extension API, Substrait & plan IRs, table functions, client connections & config scope, aggregation & GROUP BY, morsel-driven parallelism, query startup & the thread model, hash join build/probe, sorting & ORDER BY, Parquet, Iceberg, MVCC, bloom filters, NUMA. One topic per file, each tagged with the onboarding week to **prime around** — and arranged into a "why GPU-native SQL" reading thread (see its README). |
| [file-maps/](file-maps/README.md) | **Index of the per-file maps + the orchestration-spine call walk** stitching the four doorway/lifecycle/engine maps into one end-to-end control-flow trace. |
| [file-maps/sirius_extension.md](file-maps/sirius_extension.md) | `src/sirius_extension.cpp` — the doorway: extension load (Step 0) + the explicit `gpu_execution` path (Step 1b). |
| [file-maps/transparent/sirius_optimizer_extension.md](file-maps/transparent/sirius_optimizer_extension.md) | The **primary** doorway (Steps 1–2): plain-SQL interception via optimizer hooks + `PhysicalSiriusExecution`. Covers both `src/transparent/*.cpp`; converges with the explicit path on `sirius_interface`. |
| [file-maps/sirius_interface.md](file-maps/sirius_interface.md) | `src/sirius_interface.cpp` (Steps 3 & 9): query-lifecycle call graph, the `sirius_active_query` state machine, the DuckDB methods it forks. **Where both doorways converge.** |
| [file-maps/sirius_engine.md](file-maps/sirius_engine.md) | `src/sirius_engine.cpp` (Steps 4 & 5): build pipelines + launch into the scheduler; the orchestration core. *read closely* |
| [file-maps/sirius_context.md](file-maps/sirius_context.md) | `src/sirius_context.hpp` (Day 3): the ownership hierarchy + transparent-execution interceptor; the `"sirius_state"` target. *read closely* |
| [file-maps/planner/sirius_physical_plan_generator.md](file-maps/planner/sirius_physical_plan_generator.md) | `src/planner/sirius_physical_plan_generator.cpp`: logical→physical translation switch (the dispatcher); Sirius's supported-feature boundary. |
| [file-maps/planner/plan-builders.md](file-maps/planner/plan-builders.md) | `src/planner/sirius_plan_*.cpp` — the ~18 per-node builders the dispatcher calls (one `create_plan(LogicalX&)` each): construct each Sirius operator and call `from_duckdb` to attach expression ASTs. The producer call sites for the expression layer. |
| [file-maps/planner/sirius_plan_aggregate.md](file-maps/planner/sirius_plan_aggregate.md) | `src/planner/sirius_plan_aggregate.cpp` — heavyweight builder: grouped-vs-ungrouped choice, HUGEINT→BIGINT downcast, expression hoisting, perfect-hash/partitioned eligibility scaffolding. |
| [file-maps/planner/sirius_plan_comparison_join.md](file-maps/planner/sirius_plan_comparison_join.md) | `src/planner/sirius_plan_comparison_join.cpp` — heavyweight builder: hash-vs-NLJ strategy, the `prove_unique_columns` analysis → `unique_build_keys` (distinct-hash-join fast path), and the dynamic-filter pushdown hook. |
| [file-maps/op/sirius_physical_operator.md](file-maps/op/sirius_physical_operator.md) | `src/op/sirius_physical_operator.cpp`: the base operator interface (execute/sink/source/build_pipelines/task-hint) every operator implements. |
| [file-maps/op/sirius_physical_limit.md](file-maps/op/sirius_physical_limit.md) | `src/op/sirius_physical_limit.cpp` (warm-up): the simplest `execute()` — parallel LIMIT via atomic claim + `cudf::slice`. |
| [file-maps/op/sirius_physical_duckdb_scan.md](file-maps/op/sirius_physical_duckdb_scan.md) | `src/op/sirius_physical_duckdb_scan.cpp` (tiny): the `DUCKDB_SCAN` source — a scan *descriptor*; execution lives in the scan executor. |
| [file-maps/op/sirius_physical_ungrouped_aggregate.md](file-maps/op/sirius_physical_ungrouped_aggregate.md) | `src/op/sirius_physical_ungrouped_aggregate.cpp`: the two-phase local→merge aggregate (`cudf::reduce`); closes the Week 2 trace. |
| [file-maps/op/sirius_physical_grouped_aggregate.md](file-maps/op/sirius_physical_grouped_aggregate.md) | `src/op/sirius_physical_grouped_aggregate.cpp` (+ merge): the actual `GROUP BY` operator (`cudf::groupby`) — where the Week 2 query's grouping happens; `HASH_GROUP_BY → PARTITION → MERGE_GROUP_BY`. |
| [file-maps/expression/ast.md](file-maps/expression/ast.md) | `src/expression/` — the **producer entry** into the expression layer: the `sirius::ast::node` AST (12 alternatives) + `from_duckdb` (DuckDB expr → Sirius AST, `nullptr`=fallback) / `to_duckdb`. What the executor/translator consume. |
| [file-maps/expression_executor/gpu_expression_executor.md](file-maps/expression_executor/gpu_expression_executor.md) | `src/expression_executor/gpu_expression_executor.cpp` (+ `specializations/`): how FILTER/PROJECTION evaluate expressions on the GPU — the "tree of AST trees," strategies, `min_ast_size`, and the (now-complete) Sirius-AST migration. |
| [file-maps/expression_executor/gpu_expression_translator.md](file-maps/expression_executor/gpu_expression_translator.md) | `src/expression_executor/gpu_expression_translator.cpp`: the *separate* DuckDB→cuDF AST translator (all-or-nothing, for mixed/inequality joins) — evaluate-here vs. translate-and-hand-off. |
| [file-maps/op/sirius_physical_top_n.md](file-maps/op/sirius_physical_top_n.md) | `src/op/sirius_physical_top_n.cpp` (+ `_merge`): the Day-3 medium operator — two-phase local→merge TOP-N, and per-case cuDF primitive selection (`top_k_order` vs. full-sort fallback). |
| [file-maps/op/sirius_physical_partition.md](file-maps/op/sirius_physical_partition.md) | `src/op/sirius_physical_partition.cpp`: the harder Day-3 alternative — hash fan-out feeding joins/group-bys; derives keys from the parent op, sizes `N`, coordinates a build/probe sibling pair. A Week 4–5 on-ramp. |
| [file-maps/pipeline/sirius_pipeline.md](file-maps/pipeline/sirius_pipeline.md) | `src/pipeline/sirius_pipeline.cpp` (Week 4, *study*): the **runnable unit** the scheduler tier revolves around — source→operators→sink (reversed at `is_ready`), the parent/dependency DAG, and the **completion bookkeeping** (task counters + `update_pipeline_status`/`notify_downstream_pipelines` that decide when a pipeline — and at `RESULT_COLLECTOR`, the query — is done). Plus the `sirius_pipeline_build_state` plan-time assembler and the created/completed race lock. |
| [file-maps/pipeline/sirius_meta_pipeline.md](file-maps/pipeline/sirius_meta_pipeline.md) | `src/pipeline/sirius_meta_pipeline.cpp` (Week 4, *read*): **execution-flow Step 4a** — carve the plan into pipelines grouped by shared sink (DuckDB meta-pipeline pattern), build-side-first. |
| [file-maps/pipeline/sirius_pipeline_converter.md](file-maps/pipeline/sirius_pipeline_converter.md) | `src/pipeline/sirius_pipeline_converter.cpp` (Week 4, *study*): **execution-flow Step 4b–4d, where most of Step 4 lives** — the 1,343-line core that splits operators into Sirius shapes (PARTITION/CONCAT/MERGE pairs, 4-phase sort) and computes repository wiring. Folds in `repository_wiring_materializer`. |
| [file-maps/pipeline/task_scheduler.md](file-maps/pipeline/task_scheduler.md) | `src/pipeline/task_scheduler.cpp` (Week 4, *study*): the concurrency core — a pull-signal matcher routing tasks to ready GPUs by device preference; owns the sub-executors. Flags the doc-vs-code RR-counter drift. |
| [file-maps/pipeline/gpu_pipeline_executor.md](file-maps/pipeline/gpu_pipeline_executor.md) | `src/pipeline/gpu_pipeline_executor.cpp` (Week 4, *study*): one per GPU — reserve→pop→dispatch loop, dispatches the task's `execute()`, and turns OOM into a resumable retry. |
| [file-maps/pipeline/gpu_pipeline_task.md](file-maps/pipeline/gpu_pipeline_task.md) | `src/pipeline/gpu_pipeline_task.cpp` (Week 4, *study*): the **resumable task the executor runs** — the per-task-device colocation contract (`prepare_for_processing`), the operator loop (`compute_task`), `oom_reschedule_exception` with a resume index, reservation sizing + the memory-history feedback loop, and `create_rescheduled_task`. Where `gpu_pipeline_executor.md` punts ("not in this file"). |
| [file-maps/creator/task_creator.md](file-maps/creator/task_creator.md) | `src/creator/task_creator.cpp` (Week 4, *read closely*): decides *what* runs next via the hint-chain recursion; builds scan vs. GPU pipeline tasks. The operator overrides carry the real logic. |
| [file-maps/op/scan/duckdb_scan_executor.md](file-maps/op/scan/duckdb_scan_executor.md) | `src/op/scan/duckdb_scan_executor.cpp` (Week 5, *read*): the host-I/O scan executor — column-builder scan tasks, a four-level result cache, memory-proportional multi-GPU targeting. |
| [file-maps/op/sirius_physical_hash_join.md](file-maps/op/sirius_physical_hash_join.md) | `src/op/sirius_physical_hash_join.cpp` (Week 5, *study*): the hardest operator — three execution modes (STANDARD / BUILD_PROBE / MIXED_JOIN), the build/probe state machine, key casts, join-type side-swapping. |
| [file-maps/memory/sirius_memory_reservation_manager.md](file-maps/memory/sirius_memory_reservation_manager.md) | `src/memory/sirius_memory_reservation_manager.cpp` (Week 6, *read closely*): the thin cuCascade bridge — three memory tiers, reserve-before-run, the defragmenter OOM policy. Honest: the real machinery is cuCascade's. |
| [file-maps/downgrade/downgrade_executor.md](file-maps/downgrade/downgrade_executor.md) | `src/downgrade/downgrade_executor.cpp` (Week 6, *read*): predicate-bounded GPU→host→disk spilling — tiered candidate fetch (repos then task queue), a monitor thread, the "free just enough" loop. |
| [file-maps/fallback.md](file-maps/fallback.md) | CPU-fallback *mechanism* (Week 6, *read*): the live whole-query replay (`run_internal_cpu_fallback_query`) + per-operator `NotImplementedException` rejects, with the S3 caveat. **Corrects the plan's stale `src/fallback.cpp` (legacy) pointer.** |

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
