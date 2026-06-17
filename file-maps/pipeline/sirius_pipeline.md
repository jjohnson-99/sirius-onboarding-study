# `pipeline/sirius_pipeline.cpp` → Concurrency-Core Map

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), tagged
**study**. Read after the things that *build* it
([`sirius_meta_pipeline.md`](sirius_meta_pipeline.md),
[`sirius_pipeline_converter.md`](sirius_pipeline_converter.md)) and alongside the things that
*run* it ([`gpu_pipeline_task.md`](gpu_pipeline_task.md),
[`gpu_pipeline_executor.md`](gpu_pipeline_executor.md)). Pair with the "Pipeline" + "Pipeline
completion" parts of `docs/super-sirius/pipeline-execution.md`.

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> study-note links are relative to this file. Line numbers accurate as of 2026-06-17 —
> re-grep before trusting one; this area moves with the scheduler.

## Where this sits

This is the **data structure at the center of the scheduler tier** — the noun every other
scheduler file is a verb about. It's referenced as a type all over the maps; this is its home.

- **[`sirius_meta_pipeline`](sirius_meta_pipeline.md) + [`sirius_pipeline_converter`](sirius_pipeline_converter.md)** (Step 4) *build* these — via the
  `sirius_pipeline_build_state` helper in this file (a `friend` that fills in the private
  `source`/`operators`/`sink`).
- **[`gpu_pipeline_task`](gpu_pipeline_task.md)** holds one (in its global state) and runs its
  operators (`get_operators()`); the [task_creator](../creator/task_creator.md) walks its
  ports (`get_next_ports_after_sink()`) to decide what to schedule next.
- **[`gpu_pipeline_executor`](gpu_pipeline_executor.md)** checks `is_pipeline_finished()` to
  detect query completion; operators (partition/concat/sort) check it on their input ports to
  know an upstream pipeline has drained.

So: **converter builds it · task runs its operators · creator reads its ports · executor &
operators read its completion flag.** Its single most important job beyond "hold the operator
chain" is **deciding when the pipeline is finished** — the logic that ultimately lets a query
report itself done.

## The key idea first: a runnable unit + its plan-time builder

Two classes live here:

1. **`sirius_pipeline`** (hpp 81) — one execution pipeline: a **source**, a chain of
   **operators**, and a **sink** (the classic DuckDB pipeline shape — see
   [push vs. pull](../../reference/explainers/push-vs-pull-execution.md)). Plus the wiring that
   makes many pipelines a DAG (`dependencies` / `parents`) and the **completion bookkeeping**
   (task counters + `update_pipeline_status`).
2. **`sirius_pipeline_build_state`** (hpp 46, cpp 246–317) — a small **plan-time** helper that
   *constructs* pipelines. It's a `friend`, so it reaches in and sets the private
   `source`/`sink`/`operators`. You'll see it used from the converter/meta-pipeline, not at run
   time.

```
sirius_pipeline
  source  ─▶ [operators…] ─▶ sink            (one pipeline)
     │                         │
     │ dependencies            │ parents (weak) ──▶ downstream pipelines
     ▼                         ▼
  built by sirius_pipeline_build_state (plan time)   completion → update_pipeline_status (run time)
```

## Anatomy & `is_ready()` — note the operator reversal

`source`/`sink` are `optional_ptr` (may be null — e.g. a sink-less leaf); `operators` is a
`vector` of operator references (cpp 194). **`is_ready()` (cpp 129) reverses `operators`** in
place (cpp 133): pipelines are *built* sink-first (the converter appends as it walks down) but
must *run* source-first, so the one-shot `is_ready` flips them. After that, `operators[0]` is
the first op to execute — which is why `update_pipeline_status` treats `operators[0]` as the
"first node."

`reset()` / `reset_sink()` / `reset_source()` (cpp 92–127) are mostly **hollowed out** — their
state-init bodies are commented (Sirius manages operator state differently than DuckDB), so
they survive as near-no-ops from the inherited shape.

## The pipeline DAG: dependencies, parents, consumers, ports

Pipelines form a dependency graph (build-side pipelines must finish before the pipelines that
consume them):

- **`add_dependency(p)` (cpp 136)** — records `p` as a dependency *and* pushes `this` onto
  `p->parents` as a **`weak_ptr`** (cpp 141), so the graph has both directions without a
  reference cycle.
- **`get_parents()` (cpp 202)** — locks the weak parents into live pointers (drops any expired).
- **`get_output_consumers()` (cpp 464)** — each parent's **source** operator: "who consumes my
  output." This is the list the executor/`notify_downstream` schedule next.
- **`get_next_ports_after_sink()` (cpp 62)** — the sink's downstream **ports**, with a
  special-case unwrap for `LEFT_DELIM_JOIN` / `RIGHT_DELIM_JOIN` composite sinks (cpp 72/79).
  Used widely — [`task_creator`](../creator/task_creator.md), partition/concat operators, the
  plan printer — to traverse the port graph that connects pipelines.

## Completion machinery (the heart of this file)

How does a pipeline — and ultimately the query — know it's done? Through per-pipeline **task
counters** and a guarded status check.

- **`mark_task_created()` (cpp 408) / `mark_task_completed()` (cpp 427)** bump
  `tasks_created` / `tasks_completed` (atomics). The task object brackets these (its ctor/dtor —
  see [`gpu_pipeline_task.md`](gpu_pipeline_task.md)). `mark_task_created` also starts the
  pipeline's NVTX range on the first task; `mark_task_completed` carries a **sanity warning**
  (cpp 449) if a task finishes *after* the pipeline was already considered done (a "we declared
  completion too early" alarm).
- **`update_pipeline_status()` (cpp 361)** is the decision, under `_status_mutex`: the pipeline
  is finished when **(a)** an operator's limit is exhausted, *or* **(b)** the first node's
  source pipeline is finished **and** all its ports are empty — **and** in either case
  `tasks_created == tasks_completed` (cpp 391). On finish it sets `pipeline_finished`, calls
  `finalize_operator()` on every operator (cpp 393), then — **outside the lock** (cpp 405) —
  `notify_downstream_pipelines()`.
- **`notify_downstream_pipelines()` (cpp 328)** — if the sink is the `RESULT_COLLECTOR` this is
  the terminal pipeline: **return early** (cpp 334) to avoid racing engine teardown after the
  query future is signalled. Otherwise schedule each output consumer via the task_creator
  (cpp 342) and call `update_pipeline_status(false)` on each parent (cpp 351), propagating
  completion up the DAG.
- **`is_pipeline_finished()` (cpp 319)** just reads the atomic — the cheap query the
  [executor](gpu_pipeline_executor.md) uses at completion check, and that operators use on their
  input ports.

### The created-vs-completed race, and the lock that closes it

There's a subtle race: `update_pipeline_status` could observe `tasks_created == tasks_completed`
*while a new task is mid-creation* (port already popped, counter not yet bumped) and wrongly
declare the pipeline done. **`get_task_creation_lock()` (cpp 356)** hands out a lock on
`_status_mutex` that the [task_creator](../creator/task_creator.md) holds across "pop the port
data **and** call `mark_task_created()`", so the status check can't slip in between. This is the
same race the creator's "`mark_task_created` *before* popping" ordering also guards — two
matching halves of one invariant.

## `sirius_pipeline_build_state` — the plan-time builder (cpp 246)

The `friend` that assembles pipelines during Step 4 (used from the converter / meta-pipeline):
`set_pipeline_source` (246), `set_pipeline_sink` (253, also sets `base_batch_index` from the
sink-pipeline count), `add_pipeline_operator` (267), `set_pipeline_operators` (286), and
**`create_child_pipeline` (293)** — when a second source operator is found, fork a child
pipeline sharing the same sink and the operators up to that source (multi-source pipelines). It
also carries `delim_join_dependencies` / `cte_dependencies` maps (hpp 53/57) for the
duplicate-eliminated-join and materialized-CTE scan wiring.

## Batch indexes & order (lighter — skim first pass)

`register_new_batch_index` (cpp 217) / `update_batch_index` (cpp 225) maintain a `multiset` of
active batch indexes (with `base_batch_index = BATCH_INCREMENT * sink_pipeline_count`), used to
track ordered progress when multiple pipelines share a source. `is_order_dependent()` (cpp 45)
reports whether any operator preserves order (source `FIXED_ORDER`, an op's `operator_order`, or
an order-dependent sink under `preserve_insertion_order`) — consulted when order matters (e.g.
ORDER BY paths).

## Honesty note — vestigial methods

`schedule()` (hpp 99), `schedule_parallel()` (hpp 216), `launch_scan_tasks()` (hpp 214), and
`schedule_sequential_task()` (hpp 213) are **declared but never defined anywhere** in the tree —
leftovers from DuckDB's own pipeline-scheduling model that Sirius's
[scheduler tier](task_scheduler.md) replaced. Likewise several methods are commented out
(`to_string`, `print`, `get_all_operators`, `get_progress`). Don't go hunting for their bodies;
the real scheduling lives in the creator/scheduler/executor.

## Methods

| Member (signature) | Line | Role |
|---|---|---|
| `sirius_pipeline(build_context&)` | cpp 40 | construct empty; stores plan-time `build_ctx_`. |
| `void is_ready()` | cpp 129 | one-shot: **reverse `operators`** (built sink-first → run source-first). |
| `void add_dependency(shared_ptr&)` | cpp 136 | record dependency + push `this` as a weak `parent`. |
| `get_operators()` | cpp 185 / 191 | the operator chain (run order, post-`is_ready`). |
| `get_parents()` | cpp 202 | lock weak parents into live pointers. |
| `get_output_consumers()` | cpp 464 | parents' source operators — who consumes this output. |
| `get_next_ports_after_sink()` | cpp 62 | sink's downstream ports; unwraps delim-join composites. |
| `void update_pipeline_status(bool)` | cpp 361 | the finish decision (counters + source-done + ports-empty), under `_status_mutex`; finalizes ops; notifies downstream. |
| `void notify_downstream_pipelines(bool)` | cpp 328 | schedule consumers + propagate to parents; early-return on `RESULT_COLLECTOR`. |
| `bool is_pipeline_finished()` *(virtual)* | cpp 319 | read the `pipeline_finished` atomic. |
| `void mark_task_created()` / `mark_task_completed()` | cpp 408 / 427 | bump task counters (task ctor/dtor call these); NVTX + the too-early-completion warning. |
| `unique_lock get_task_creation_lock()` | cpp 356 | the lock the creator holds across port-pop + `mark_task_created` (closes the created/completed race). |
| `bool is_order_dependent()` | cpp 45 | does any operator/source/sink require order preservation? |
| `register_new_batch_index()` / `update_batch_index()` | cpp 217 / 225 | active-batch-index multiset for ordered multi-pipeline sources. |
| `sirius_pipeline_build_state::set_pipeline_{source,sink,operators}` | cpp 246 / 253 / 286 | **plan-time** mutators (friend reaches into privates). |
| `sirius_pipeline_build_state::create_child_pipeline(...)` | cpp 293 | fork a child pipeline at a second source (shared sink). |
| *(declared, undefined)* `schedule` / `schedule_parallel` / `launch_scan_tasks` | hpp 99 / 216 / 214 | **vestigial** — see honesty note. |

## Types fundamental to *this* file

- **`sirius_physical_operator`** *(Sirius op; [base map](../op/sirius_physical_operator.md))* —
  what `source`/`operators`/`sink` point at; provides `get_next_ports_after_sink`,
  `finalize_operator`, `all_ports_empty`, `is_source_pipeline_finished`. **Think:** "the nodes
  the pipeline strings together."
- **`sirius_pipeline_build_state`** *(this file)* — plan-time assembler. **Think:** "the only
  thing allowed to fill in a pipeline's source/operators/sink."
- **`pipeline_build_context`** *(Sirius; `pipeline/pipeline_build_context.hpp`)* — the plan-time
  context the pipeline keeps (`build_ctx_`, e.g. `preserve_insertion_order`) instead of a live
  `sirius_engine&`. **Think:** "the build-time config snapshot."
- **`next_port_info` / ports** *(Sirius op)* — the connectors between pipelines that the creator
  polls; see [push vs. pull](../../reference/explainers/push-vs-pull-execution.md). **Think:**
  "the wires the DAG is made of."
- **`optional_ptr<T>` / `weak_ptr`** *(DuckDB / std)* — nullable source/sink; weak `parents` to
  avoid a cycle with `shared_ptr` `dependencies`. **Think:** "may-be-absent node; non-owning
  back-edge."

## Takeaway

`sirius_pipeline` is the **runnable unit** the whole scheduler tier revolves around: a
source→operators→sink chain (operators reversed at `is_ready`), wired into a parent/dependency
**DAG**, carrying the **completion bookkeeping** (task counters + `update_pipeline_status` +
`notify_downstream_pipelines`) that decides when a pipeline — and, at the `RESULT_COLLECTOR`,
the query — is done. Its `sirius_pipeline_build_state` half is the plan-time assembler the
converter uses. With this, the scheduler tier is fully mapped: [meta-pipeline](sirius_meta_pipeline.md)
/ [converter](sirius_pipeline_converter.md) build pipelines → [creator](../creator/task_creator.md)
turns ports into tasks → [scheduler](task_scheduler.md) routes them → [executor](gpu_pipeline_executor.md)
dispatches → [task](gpu_pipeline_task.md) runs the operators → **pipeline** counts them home.
