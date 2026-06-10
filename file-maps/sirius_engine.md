# `sirius_engine.cpp` → Execution-Flow Map

Companion for Week 2, Days 1–2 of [`onboarding-path.md`](../onboarding-path.md),
tagged **read closely** — the orchestration core. Read `src/sirius_engine.cpp` (+
`src/include/sirius_engine.hpp`) with `docs/super-sirius/execution-flow.md` **and**
`docs/super-sirius/physical-plan-generation.md` open. This map goes deep.

> Source/doc paths are relative to the Sirius repo root (`../sirius/` beside this
> folder). Line numbers accurate as of 2026-06-10; re-grep if the file moved.

## Where this sits

[`sirius_interface.md`](sirius_interface.md) (the lifecycle conductor) drives this
file: it constructs a `sirius_engine`, then calls three verbs on it —
`initialize()`, `execute()`, `get_result()`. **This is the file where those verbs
live.** The engine itself contains almost no GPU logic; it is the **builder +
launcher**:

- **build** — turn the physical-operator tree (from
  [`planner/sirius_physical_plan_generator.md`](planner/sirius_physical_plan_generator.md))
  into runnable *pipelines* (execution-flow **Step 4**),
- **launch** — hand those pipelines to the `task_scheduler` and block until done
  (**Step 5**),
- **yield** — pull the materialized result out of the RESULT_COLLECTOR (**Step 9**).

The heavy lifting of *how* pipelines are split is delegated to
`sirius_pipeline_converter` (Week 4–5 reading); the engine is the ~35-line
orchestrator around it.

## The three verbs (call them in this order)

| Function (signature) | Line | Doc step / role |
|---|---|---|
| `void initialize(unique_ptr<op::sirius_physical_operator> plan)` | 140 | **Step 4 entry.** Marks the query `planning()`, `reset()`s prior state, takes ownership of the plan (`sirius_owned_plan`), pre-fetches Iceberg delete data, then calls `initialize_internal`. |
| `void initialize_internal(op::sirius_physical_operator& plan)` | 353 | **Step 4 proper.** The build pipeline (see breakdown below). Ends with `new_scheduled` = the runnable pipelines. |
| `void execute()` | 154 | **Step 5.** Hands `new_scheduled` to `SiriusContext::create_query`, calls `task_scheduler.start_query()` → a `future`, then **blocks on `future.get()`**. On any exception, calls `drain_after_error()` before rethrowing (prevents use-after-free of repositories). |
| `unique_ptr<QueryResult> get_result()` | 129 | **Step 9.** Casts the plan root to `sirius_physical_materialized_collector` and returns its materialized result. Called by `sirius_interface::fetch_result_internal`. |

### `initialize_internal()` step by step (lines 353–404)

This is the part to read slowly — it's the whole of Step 4 in ~50 lines:

1. **Get config** (355–361) — pull `operator_params` off the `SiriusContext`
   (`registered_state->Get<SiriusContext>("sirius_state")`, the same keyed lookup you
   saw in the extension/interface files).
2. **Build the meta-pipeline tree** (366–377) — create a `sirius_meta_pipeline` root
   and call `root_pipeline->build(plan)` then `->ready()`. This recurses through every
   operator's `build_pipelines()` (the base implementation is in
   [`op/sirius_physical_operator.md`](op/sirius_physical_operator.md)), producing
   DuckDB-style pipelines (separate source/operators/sink). *This is plan-gen doc
   Part 2.*
3. **Convert / split** (380–383) — `sirius_pipeline_converter(...).convert(root)` does
   the Sirius-specific splitting: injecting PARTITION/CONCAT/MERGE operators at
   pipeline boundaries and finalizing each pipeline's operator list. *This is plan-gen
   doc Part 3.* (Deferred to Week 4–5; just know the engine launches it here.)
4. **Materialize wiring** (386–387) — turn the converter's plan-time wiring
   descriptors into real `shared_data_repository` objects + ports. *Plan-gen doc
   Part 4.*
5. **Stash results** (389–403) — `new_scheduled` (the runnable pipelines),
   `new_pipeline_breakers` (injected operators), and a debug plan dump.

## Supporting machinery

| Function (signature) | Line | Role |
|---|---|---|
| `unique_ptr<op::sirius_physical_operator> construct_sirius_specific_operator(op::sirius_physical_operator*)` | 204 | **Read this one closely.** Maps a "phase-1" operator to its injected "phase-2" partner: `TABLE_SCAN`→ parquet/iceberg/**duckdb** scan (by `function.name`: `seq_scan`→`DUCKDB_SCAN`), `HASH_GROUP_BY`→`MERGE_GROUP_BY`, `ORDER_BY`→`MERGE_SORT`, `TOP_N`→`MERGE_TOP_N`, **`UNGROUPED_AGGREGATE`→`MERGE_AGGREGATE`**. This is where the two-phase operator pairs (local + merge) are born — directly relevant to the aggregate you read on Days 4–5. |
| `sirius_engine(ClientContext&, sirius_interface&)` | 83 | Ctor — wires telemetry (`quent` query/group handles). Skim. |
| `~sirius_engine()` | 104 | Telemetry teardown. |
| `void reset()` | 106 | Clears all pipeline/plan state between queries. |
| `void execute()` post-run check | 188–201 | Warns about operators that never got `finalized` — a debugging aid, not core logic. |
| `void prefetch_iceberg_delete_data(...)` / `construct_iceberg_scan_operator(...)` / `resolve_iceberg_table_path(...)` | 255–351 | **Iceberg-only.** Pre-materializes delete files before operator IDs are assigned. *Skip unless working on Iceberg* — orthogonal to the target query. |
| `bool has_result_collector()` | 124 | True iff the plan root is a `RESULT_COLLECTOR`. |

## Why `initialize()` and `execute()` are separate

Same two-phase logic as the interface: `initialize()` builds everything but runs
nothing; `execute()` runs. This mirrors `sirius_interface`'s pending/execute split
(the interface's `sirius_pending_statement_internal` calls `engine.initialize()`;
`sirius_execute_pending_query_result` calls `engine.execute()`). Build-time errors
(unsupported plan shapes) surface from `initialize`; runtime errors from `execute`.

## Types fundamental to *this* file

Recurring DuckDB types (`ClientContext`, `QueryResult`) →
[`duckdb-types-glossary.md`](../reference/duckdb-types-glossary.md). The ones that
matter specifically here:

- **`op::sirius_physical_operator`** *(Sirius base class)* — the node type of the
  physical plan the engine consumes. Owned via `sirius_owned_plan`; the live pointer
  is `sirius_physical_plan`. Full treatment in
  [`op/sirius_physical_operator.md`](op/sirius_physical_operator.md). **Think:** the
  GPU plan tree the engine turns into pipelines.
- **`pipeline::sirius_meta_pipeline` / `sirius_pipeline`** *(Sirius types)* — a
  *meta-pipeline* is a group of pipelines sharing a sink; a *pipeline* is an ordered
  operator list that becomes the unit of GPU scheduling. The engine builds the former
  and emits the latter into `new_scheduled`. Deep detail is Week 4 reading
  (`pipeline-execution.md`); for now: **pipeline = the schedulable chunk of work.**
- **`SiriusContext`** *(Sirius, a DuckDB `ClientContextState`)* — reached via
  `context.registered_state->Get<SiriusContext>("sirius_state")`. The engine borrows
  the config, the `task_scheduler`, and the data-repository manager from it. Full
  treatment in [`sirius_context.md`](sirius_context.md). **Think:** the owner of every
  subsystem the engine calls into.
- **`sirius_pipeline_converter`** *(Sirius type)* — does the actual pipeline splitting
  the engine delegates to. **Think:** the engine's heavy machinery, read later.

## Takeaway into the next files

The engine consumes a plan (from the **plan generator**, next file) and produces
pipelines that it launches into the **`task_scheduler`** (Week 4). The operator
two-phase pairing it sets up in `construct_sirius_specific_operator` is exactly what
makes the **ungrouped aggregate** (Days 4–5) split into local + merge. So the natural
reading order from here is: plan generator (what builds the tree the engine
consumes) → `sirius_context` (what owns the scheduler the engine launches into) →
the operators themselves.
