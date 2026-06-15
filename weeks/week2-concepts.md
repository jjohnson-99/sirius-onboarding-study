# Week 2 Concepts — Tracing One Query End-to-End

The Week 2 checkpoint in [`onboarding-path.md`](../onboarding-path.md):

> Can point at the file + function for each stage of a query's life. **Ready for a
> small contribution.**

This is the synthesis. Week 1 ([`week1-concepts.md`](week1-concepts.md)) gave you the
vocabulary (physical plan, operator, pipeline, hash join). Week 2 follows **one real
query through the actual source**. Each stage links to a per-file map in
[`file-maps`](../file-maps) that goes deeper; this doc is the through-line that
connects them, mapped onto `docs/super-sirius/execution-flow.md`.

## The query

```sql
SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY l_returnflag;
```

A scan + aggregate — no join — so it exercises the spine without the hardest operator.
(`lineitem` is a DuckDB table, so the scan resolves to `DUCKDB_SCAN`.)

## The end-to-end trace

Eight stages, from SQL string to result. File:function references are the "point at
it" the checkpoint asks for.

### 1. Doorways — SQL enters Sirius
**File maps:** [`sirius_extension.md`](../file-maps/sirius_extension.md) (Step 0 + explicit), [`transparent/sirius_optimizer_extension.md`](../file-maps/transparent/sirius_optimizer_extension.md) (primary) · **doc:** Steps 1 / 1b / 2

First, `LoadInternal` in `sirius_extension.cpp` bootstraps the extension (**Step 0**) —
registering config, the optimizer hooks, and the `gpu_execution` function. Then there
are **two doorways**, and the key fact is that **they converge** on one engine entry
point, `sirius_interface::sirius_execute_query`:

- **Explicit `CALL gpu_execution(...)`** (read first — cleanest linear trace):
  `GPUExecutionBind` (parse/optimize/plan) → `GPUExecutionFunction` (run) →
  `sirius_execute_query`.
- **Transparent / plain SQL** (the production default): optimizer hooks capture the
  optimized logical plan → `SiriusContext::OnFinalizePrepare` swaps DuckDB's physical
  plan for a `PhysicalSiriusExecution` → its `GetDataInternal` calls
  `sirius_execute_query`.

So everything from Stage 3 onward is **shared by both paths** — the doorway is the
only difference. ("Legacy" on Step 1b means the explicit *invocation* is non-default,
not that the code is dead; the truly-legacy `gpu_processing` engine is a separate
thing behind `#ifdef SIRIUS_ENABLE_LEGACY`.) Unsupported queries silently **fall
back** to DuckDB CPU at the plan-generation step.

### 2. Translation — logical plan → Sirius physical plan
**File map:** [`planner/sirius_physical_plan_generator.md`](../file-maps/planner/sirius_physical_plan_generator.md) · **doc:** Part 1 of physical-plan-generation

`create_plan()` walks DuckDB's `LogicalOperator` tree through a big `switch`:
`LOGICAL_GET → TABLE_SCAN`, `LOGICAL_AGGREGATE_AND_GROUP_BY → HASH_GROUP_BY`. Any
unhandled node throws → fallback. Output: a `sirius_physical_operator` tree (the
*how-on-GPU* plan).

### 3. Lifecycle setup — the conductor
**File map:** [`sirius_interface.md`](../file-maps/sirius_interface.md) · **doc:** Step 3

`sirius_execute_query` runs DuckDB's two-phase contract (a hand-rolled fork of
`ClientContext`): the **pending phase** builds a `sirius_engine`, creates the
RESULT_COLLECTOR sink, and calls `engine.initialize()`; the **execute phase** calls
`engine.execute()` then fetches the result. All per-query state lives in the single
`sirius_active_query` member.

### 4. Build — plan → pipelines
**File map:** [`sirius_engine.md`](../file-maps/sirius_engine.md) · **doc:** Step 4 / plan-gen Parts 2–4

`engine.initialize_internal()` builds a meta-pipeline tree (each operator's
`build_pipelines()`), then `sirius_pipeline_converter` splits it: the aggregate becomes
`HASH_GROUP_BY` (local) → PARTITION → `MERGE_GROUP_BY` (final), wired with barriers.
The result is `new_scheduled` — the runnable pipelines.

### 5. Ownership — who runs it
**File map:** [`sirius_context.md`](../file-maps/sirius_context.md) · **doc:** architecture-overview (Ownership Hierarchy)

`engine.execute()` reaches the `SiriusContext` via
`registered_state->Get<SiriusContext>("sirius_state")`, calls `create_query(new_scheduled)`,
and hands off to the **`task_scheduler`**. `SiriusContext` is the owner of every
subsystem (scheduler, memory manager, scan manager, task creator…) — the root of the
runtime, attached to the DuckDB connection.

### 6. Source — data enters
**File maps:** [`op/sirius_physical_operator.md`](../file-maps/op/sirius_physical_operator.md) (base), [`op/sirius_physical_duckdb_scan.md`](../file-maps/op/sirius_physical_duckdb_scan.md) · **doc:** Step 6

The `DUCKDB_SCAN` source *describes* the `lineitem` scan; the **scan executor** (Week 5)
actually reads it, batch by batch, onto the GPU, setting `exhausted` when done. Each
batch is a `pipelineable_operator_data` wrapping a `cudf::table`.

### 7. Compute — the GPU does the work
**File maps:** [`op/sirius_physical_operator.md`](../file-maps/op/sirius_physical_operator.md), [`op/sirius_physical_limit.md`](../file-maps/op/sirius_physical_limit.md) (the `execute()` template), [`op/sirius_physical_ungrouped_aggregate.md`](../file-maps/op/sirius_physical_ungrouped_aggregate.md) (the simpler sibling you read), [`op/sirius_physical_grouped_aggregate.md`](../file-maps/op/sirius_physical_grouped_aggregate.md) (**what this query uses**) · **doc:** Steps 7–8

The GPU pipeline executor calls each operator's `execute()` on a CUDA stream. Our query has a
`GROUP BY`, so its aggregate is the **grouped** path — a three-operator chain
(`physical-plan-generation.md`): `HASH_GROUP_BY` → `PARTITION` → `MERGE_GROUP_BY`:

- **`HASH_GROUP_BY`** (`sirius_physical_grouped_aggregate`): for each scanned batch,
  `cudf::groupby` buckets rows by `l_returnflag` and sums `l_quantity` **within that batch** —
  *this is where the grouping happens* — producing partial per-group rows.
- **`PARTITION`**: hash-partitions those partials **by `l_returnflag`** so the same group from
  different batches colocates.
- **`MERGE_GROUP_BY`**: combines the partials per group → the final one-row-per-group answer.

So `SUM(l_quantity)` *per group* is computed by `cudf::groupby` (local) then finalized in the
merge. ⚠️ The **ungrouped** operator you read closely (`cudf::reduce`, **one global group, no
`PARTITION`**) is the simpler *no-`GROUP BY`* sibling — same two-phase local→merge plumbing,
minus the grouping. You learned the plumbing there; the grouped operator adds `cudf::groupby`
+ partition-by-key (see its [map](../file-maps/op/sirius_physical_grouped_aggregate.md) and
the [aggregation explainer](../reference/explainers/aggregation-and-group-by.md)).

The task creator (Week 4) keeps scheduling downstream operators via the
`get_next_task_hint()` port/barrier logic in the base operator.

### 8. Result — back to DuckDB
**File map:** [`sirius_interface.md`](../file-maps/sirius_interface.md) · **doc:** Step 9

The final RESULT_COLLECTOR pipeline completes; the future resolves;
`engine.get_result()` pulls the materialized `ColumnDataCollection`;
`fetch_result_internal` → `cleanup_internal` tears down `sirius_active_query`; DuckDB
hands the rows to the client.

> For the **code-level** version of this trace — the four orchestration maps' call-sequence
> trees stitched into one spine — see [`file-maps/README.md`](../file-maps/README.md).

## The one-screen mental model

```
SQL ─▶ sirius_extension ─▶ plan_generator ─▶ sirius_interface ─▶ sirius_engine ─▶ task_scheduler
        (doorway)           (logical→phys)    (lifecycle)         (build+launch)   (run)
                                                                                       │
                                                       ┌───────────────────────────────┘
                                                       ▼
                              DUCKDB_SCAN ─▶ AGGREGATE(local) ─▶ MERGE ─▶ RESULT_COLLECTOR ─▶ DuckDB
                              (scan op)      (cudf::reduce)       (combine)  (materialize)     (client)
                                   ▲                    ▲
                            SiriusContext owns the scheduler, memory, scan manager that make this run
```

## What recurs (and is worth over-learning now)

- **The `"sirius_state"` lookup** — `registered_state->Get<SiriusContext>(...)` appears
  in every layer; it's how any code reaches the runtime.
- **Two-phase everything** — interface (pending/execute), engine (initialize/execute),
  aggregates (local/merge), sorts (partition/merge). The same "set up, then run /
  compute locally, then combine" rhythm.
- **The `execute()` shape** — get `table_view` → call a `cudf::` function on the
  `stream` → re-wrap as a batch. Every operator's GPU work is a variation on this.
- **Fallback as a first-class path** — unsupported logical ops (plan generator) and
  runtime errors (interface) both route back to DuckDB CPU.

## Where the spine stops (and Weeks 4–6 pick up)

Three subsystems were *named* but not entered — they're the deep-dive weeks:
- **`task_scheduler` / `task_creator` / GPU pipeline executor** — how tasks actually get
  scheduled and run (Week 4).
- **scan executor / scan manager** — how `DUCKDB_SCAN`/parquet data really lands on the
  GPU (Week 5).
- **memory reservation manager / downgrade** — reservations and GPU→host spilling
  (Week 6).

You can now point at the file and function for each stage of the query's life — the
Week 2 checkpoint — and you're ready for a small contribution (Week 3).
