# `creator/task_creator.cpp` → Task-Creation Map

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md) (the
`src/creator/` portion), tagged **read closely**. Read with
`docs/super-sirius/task-creator.md` — that doc is unusually complete (full override table)
and this map is the code-anchored companion to it. Read after the two pipeline maps
([`../pipeline/task_scheduler.md`](../pipeline/task_scheduler.md),
[`../pipeline/gpu_pipeline_executor.md`](../pipeline/gpu_pipeline_executor.md)).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — re-grep.

> **All line numbers below refreshed for pull `d9172de6` (2026-06-15).** This file got a big
> (503-line) reorg from the scan-operator unification (`#871`) + Quent telemetry (`#809`), but
> the **hint-chain mechanism is unchanged**. Full diff:
> [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md).

## The key idea first: it answers "what becomes the next task?"

The scheduler tier has a clean division of labor:
- **`task_creator`** (this file) decides **which operator** should produce the next task,
  builds the concrete task, and dispatches it.
- **`task_scheduler`** decides **which GPU / when**.
- **`gpu_pipeline_executor`** **runs** it.

The central trick is the **hint chain**: an operator that isn't ready points *upstream* to
its producer, and the creator recurses until it finds the deepest operator that *is*
ready. This is how Sirius gets **data flowing from the deepest producers first**, honoring
pipeline dependencies — the practical expression of morsel-driven, demand-style
scheduling (prime with
[`../../reference/explainers/morsel-driven-parallelism.md`](../../reference/explainers/morsel-driven-parallelism.md)
and [`../../reference/explainers/push-vs-pull-execution.md`](../../reference/explainers/push-vs-pull-execution.md)).

## Call sequence

```
schedule(operator*)                              :196   (called by engine kickoff + by GPU exec for consumers)
└─ _task_creation_queue.push(request)
                                                       ── separate thread ──
manager_loop()                                   :203
└─ loop while running:
   ├─ _thread_pool.reserve()                            wait for a creation slot (bounded pool)
   ├─ req = _task_creation_queue.pop()
   ├─ node = get_operator_for_next_task(req.node) :220  ─ follow the hint chain (recursion) ─┐
   ├─ if node == nullptr: continue                                                            │
   └─ build + dispatch the right task kind:                                                   │
        ├ DUCKDB_SCAN  → duckdb_scan_task   ─▶ scan_executor                                  │
        ├ PARQUET_SCAN → one parquet_scan_task per row-group partition ─▶ scan_executor       │
        └ GPU operator → loop while ports non-empty:                                          │
              mark_task_created()   (BEFORE popping — closes a finished/!finished race)        │
              data = node->get_next_task_input_data()                                          │
              if data ─▶ gpu_pipeline_task ─▶ task_scheduler->schedule()                       │
              else      mark_task_completed()                                                  │
                                                                                              │
get_operator_for_next_task(node)                 :135  ◀───────────────────────────────────────┘
├─ PARQUET_SCAN special-case: ready iff has_more_partitions()
├─ hint = node->get_next_task_hint()                    (base impl in sirius_physical_operator.cpp)
│     ├ READY               → return hint.producer      (make a task from this op)
│     └ WAITING_FOR_INPUT   → recurse on hint.producer  :151  (go deeper upstream)
└─ DUCKDB_SCAN special-case: if drained/!can_create_more → nullptr
```

The `mark_task_created()` **before** `get_next_task_input_data()` is a deliberate race
fix: it prevents the pipeline from looking "finished" in the gap between checking for data
and actually creating the task.

## The two methods the whole design hangs on (defined on operators, not here)

These live on `sirius_physical_operator` (base) and are overridden per operator — this is
where most operator-specific scheduling behavior actually lives. See
[`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md) for the base.

- **`get_next_task_hint()`** → `READY` (make a task) or `WAITING_FOR_INPUT_DATA` (+ a
  `producer` to chase). Base impl: respects `FULL` barriers (wait for the whole upstream
  pipeline), then `PARTIAL`, else `READY`/`nullopt`.
- **`get_next_task_input_data()`** → pops the actual input batches from the operator's
  ports. Base impl: one batch per port.

You already met three overrides in Week 3:
- **PARTITION** ([`../op/sirius_physical_partition.md`](../op/sirius_physical_partition.md))
  — sibling-coordinated partition-count determination.
- **TOP_N_MERGE** ([`../op/sirius_physical_top_n.md`](../op/sirius_physical_top_n.md)) and
  **GROUPED_AGGREGATE_MERGE**
  ([`../op/sirius_physical_grouped_aggregate.md`](../op/sirius_physical_grouped_aggregate.md))
  — "drain all batches of one partition per task."

The big one for Week 5 is **HASH_JOIN BUILD_PROBE**, whose
`get_next_task_input_data_for_build_probe()` drives a build/probe state machine — covered
in [`../op/sirius_physical_hash_join.md`](../op/sirius_physical_hash_join.md). The doc's
"Override Summary Table" is the best one-screen index of all of them.

## Lifecycle methods (cpp)

| Method (signature) | Line | Role |
|---|---|---|
| `task_creator(config, …)` | 44 | owns a `bounded_thread_pool` for parallel task creation. |
| `void prepare_for_query(query)` | 82 | build the per-operator global-state maps (scan batching, parquet partitions, gpu pipeline state); parquet metadata can be **rebound** for warm re-runs. |
| `void schedule(operator*)` | 196 | enqueue a creation request (entry point for engine + downstream consumers). |
| `void manager_loop()` | 203 | the create-and-dispatch loop above. |
| `operator* get_operator_for_next_task(node)` | 135 | the recursive hint chain. |
| `void drain_pending_tasks()` | 109 | drain queue + wait for in-flight creation lambdas (part of `drain_after_error`). |
| `void reset(keep_parquet_metadata)` | 127 | clear per-query state between queries (optionally preserving parquet metadata for the cache). |

## Global state maps (built in `prepare_for_query`, guarded by `_global_state_mutex`)

| Map | Keyed by | Holds |
|---|---|---|
| `_scan_operator_global_state_map` | operator id | DuckDB scan batching state |
| `_parquet_scan_operator_global_state_map` | operator id | parquet metadata + row-group partitions (survives warm runs) |
| `_gpu_operator_global_state_map` | operator id | shared pipeline state |

These exist because tasks for the same operator must share progress state (which partition
is next, whether the scan is drained) across the many tasks that operator spawns.

## Types fundamental to *this* file

- **`task_creation_hint` / `TaskCreationHint`** *(Sirius; in
  `sirius_physical_operator.hpp`)* — `{READY, WAITING_FOR_INPUT_DATA}` + a `producer*`.
  **Think:** "an operator's answer to 'can you run? if not, who feeds you?'"
- **`duckdb_scan_task` / `parquet_scan_task` / `gpu_pipeline_task`** *(Sirius)* — the three
  concrete task kinds this file builds and dispatches. **Think:** "the creator's three
  output shapes; scans go to the scan executor, everything else to the scheduler."
- **`can_create_more_tasks()` / `has_processed_all_tasks()`** *(Sirius; operator hooks)* —
  exhaustion signals (base throws; sources override). **Think:** "is this source out of
  work / are its tasks all done."
- **`exec::bounded_thread_pool` / the creation queue** — same throttling primitive as the
  GPU executor; creation itself is parallelized.

## Takeaway

`task_creator` is the demand-driven brain: a hint-chain recursion finds the deepest ready
operator, then it builds the matching task kind and ships scans to the scan executor and
GPU work to the scheduler. The *interesting* logic is distributed onto the operators'
`get_next_task_hint()` / `get_next_task_input_data()` overrides — read this file together
with the doc's override table, then go see the most elaborate override in
[`../op/sirius_physical_hash_join.md`](../op/sirius_physical_hash_join.md). The data those
tasks move is the subject of Week 5's scan +
[`../op/scan/duckdb_scan_executor.md`](../op/scan/duckdb_scan_executor.md).
