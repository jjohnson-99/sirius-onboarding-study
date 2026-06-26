# `sirius_engine.cpp` → Execution-Flow Map

Companion for Week 2, Days 1–2 of [`onboarding-path.md`](../onboarding-path.md),
tagged **read closely** — the orchestration core. Read `src/sirius_engine.cpp` (+
`src/include/sirius_engine.hpp`) with `docs/super-sirius/execution-flow.md` **and**
`docs/super-sirius/physical-plan-generation.md` open. This map goes deep.

> Source/doc paths are relative to the Sirius repo root. Line numbers accurate as of 2026-06-10; re-grep if the file moved.

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

The heavy lifting of *how* pipelines are built and split is delegated to `src/pipeline/`:
the meta-pipeline tree ([`pipeline/sirius_meta_pipeline.md`](pipeline/sirius_meta_pipeline.md),
Step 4a) and **the converter**
([`pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md), Step 4b–4d —
the 1,343-line core where the splitting happens). The engine is the ~50-line orchestrator
around them. *(If you're wondering "where does Step 4 actually live?" — it's the converter,
not this file.)*

## Call sequence

The three verbs are called by `sirius_interface` (see
[`sirius_interface.md`](sirius_interface.md)); `─▶` leaves this file:

```
initialize(plan)                              :143   ◀─ Step 4 entry
├─ reset()                                    :109
├─ prefetch_iceberg_delete_data(plan)         :293     (iceberg-only; skip on first read)
└─ initialize_internal(plan)                  :350   ══ BUILD PIPELINES
   ├─ root_pipeline->build(plan) / ready()    :372   ─▶ operators' build_pipelines()  · Step 4a
   │                                                   (sirius_meta_pipeline.md)
   ├─ converter.convert(root_pipeline)        :380   ─▶ sirius_pipeline_converter  (split + wire) · Step 4b–4d
   │     ⋯ calls construct_sirius_specific_operator()  (converter.cpp:61)  → injects MERGE / 2nd-phase ops
   ├─ materialize_repository_wiring(…)         :383     plan-time wiring → live repositories · Step 4c
   └─ new_scheduled = …                        :386     ◀ the runnable pipelines

execute()                                     :157   ══ LAUNCH + WAIT  (Step 5)
├─ ctx->create_query(new_scheduled, …)        :168   ─▶ prepare_for_query: THE SETUP — make completion_handler, wire executors
├─ task_scheduler.start_query()               :173   ─▶ THE TRIGGER — schedule(first scan); return the future
└─ future.get()                               :175     BLOCKS this thread until completion_handler fires
      ⋯ on exception → drain_after_error()     :183     drains in-flight GPU tasks first

get_result()                                  :132   ◀─ Step 9 (called by interface's fetch)
└─ result_collector.get_result()              :139     materialized result out
```

`initialize` (build) and `execute` (launch) mirror `sirius_interface`'s pending/execute
split. The `─▶` arrows are the engine's handoffs — to the **meta-pipeline** + **converter**
(the real pipeline construction, Step 4 —
[`pipeline/sirius_meta_pipeline.md`](pipeline/sirius_meta_pipeline.md) /
[`pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md)), the
**`task_scheduler`** (Week 4), and **`SiriusContext`**; the engine itself just orchestrates.
`construct_sirius_specific_operator` lives *inside* the converter (converter.cpp:61), where
the local→merge operator pairs are born.

> **`execute()` looks like it launches the engine; it really just *triggers* one already
> running.** The scheduler's threads — the `management_eventloop`, one `gpu_pipeline_executor`
> manager loop per GPU, the task_creator loop, the scan/downgrade executors — were all spawned
> earlier at `SiriusContext::initialize()` and are sitting *blocked*, waiting for work.
> `create_query` (→ `prepare_for_query`) does the per-query setup (creates the
> `completion_handler`, wires it to the executors); `start_query()` does almost nothing —
> schedules the first scan and returns the `future`; then `execute()` **blocks on
> `future.get()`** until a worker thread signals completion. The full beginner-friendly
> walkthrough (what each thread waits on, when it wakes) is in
> [query startup & the thread model](../reference/explainers/query-startup-and-threads.md).

## The three verbs (call them in this order)

| Function (signature) | Line | Doc step / role |
|---|---|---|
| `void initialize(unique_ptr<op::sirius_physical_operator> plan)` | 143 | **Step 4 entry.** Marks the query `planning()`, `reset()`s prior state, takes ownership of the plan (`sirius_owned_plan`), pre-fetches Iceberg delete data, then calls `initialize_internal`. |
| `void initialize_internal(op::sirius_physical_operator& plan)` | 350 | **Step 4 orchestrator** (the real work is in `src/pipeline/` — see breakdown below). Ends with `new_scheduled` = the runnable pipelines. |
| `void execute()` | 157 | **Step 5.** Hands `new_scheduled` to `SiriusContext::create_query`, calls `task_scheduler.start_query()` → a `future`, then **blocks on `future.get()`**. On any exception, calls `drain_after_error()` before rethrowing (prevents use-after-free of repositories). |
| `unique_ptr<QueryResult> get_result()` | 132 | **Step 9.** Casts the plan root to `sirius_physical_materialized_collector` and returns its materialized result. Called by `sirius_interface::fetch_result_internal`. |

### `initialize_internal()` step by step (lines 350–401)

This is the part to read slowly — but note it only *orchestrates* Step 4 in ~50 lines; the
substance is in [`pipeline/sirius_meta_pipeline.md`](pipeline/sirius_meta_pipeline.md) (4a) and
[`pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md) (4b–4d):

1. **Get config** (352–358) — pull `operator_params` off the `SiriusContext`
   (`registered_state->Get<SiriusContext>("sirius_state")`, the same keyed lookup you
   saw in the extension/interface files).
2. **Build the meta-pipeline tree** (362–374) — create a `sirius_meta_pipeline` root
   and call `root_pipeline->build(plan)` then `->ready()`. This recurses through every
   operator's `build_pipelines()` (the base implementation is in
   [`op/sirius_physical_operator.md`](op/sirius_physical_operator.md)), producing
   DuckDB-style pipelines (separate source/operators/sink). *This is plan-gen doc
   Part 2.*
3. **Convert / split** (380) — `sirius_pipeline_converter(...).convert(root)` does
   the Sirius-specific splitting: injecting PARTITION/CONCAT/MERGE operators at
   pipeline boundaries and finalizing each pipeline's operator list. *This is plan-gen
   doc Part 3 / execution-flow Step 4b–4d* — the substance, fully mapped in
   [`pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md).
4. **Materialize wiring** (383) — turn the converter's plan-time wiring
   descriptors into real `shared_data_repository` objects + ports (`repository_wiring_materializer.cpp`).
   *Plan-gen doc Part 4 / Step 4c.*
5. **Stash results** (386–401) — `new_scheduled` (the runnable pipelines),
   `new_pipeline_breakers` (injected operators), and a debug plan dump.

## Supporting machinery

| Function (signature) | Line | Role |
|---|---|---|
| `unique_ptr<op::sirius_physical_operator> construct_sirius_specific_operator(op::sirius_physical_operator*)` | 204 | ⚠️ **DEAD — zero callers** (re-verified 2026-06-26). This *engine member* maps a "phase-1" operator to its "phase-2" partner (`HASH_GROUP_BY`→`MERGE_GROUP_BY`, `ORDER_BY`→`MERGE_SORT`, `TOP_N`→`MERGE_TOP_N`, `UNGROUPED_AGGREGATE`→`MERGE_AGGREGATE`, and `TABLE_SCAN`→ parquet/iceberg/duckdb scan), but **nothing calls it**. The two-phase pairs are actually born in the *converter free-function* of the same name (`sirius_pipeline_converter.cpp:61`, called at `:653/:670/:906` — see [`pipeline/sirius_pipeline_converter.md`](pipeline/sirius_pipeline_converter.md)). Its `TABLE_SCAN`→`DUCKDB_SCAN` branch is doubly dead post-`#871`: scans are rewritten to `GPU_SCAN` by `split_table_scan_source`. Read the **converter** version for the live aggregate pairing. |
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
