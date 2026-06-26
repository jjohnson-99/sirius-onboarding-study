# `downgrade/downgrade_executor.cpp` → Spilling Map

Companion for **Week 6** of [`onboarding-path.md`](../../onboarding-path.md) (`src/downgrade/`),
tagged **read** — GPU→host→disk spilling via cuCascade. Read with the "Downgrade Executor"
section of `docs/super-sirius/memory-management.md`, right after
[`../memory/sirius_memory_reservation_manager.md`](../memory/sirius_memory_reservation_manager.md).
Prime with
[`../../reference/explainers/gpu-memory-and-spilling.md`](../../reference/explainers/gpu-memory-and-spilling.md).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — re-grep.

## The key idea first: "downgrade" = move data to a cheaper tier to free a hot one

When the GPU is under memory pressure, the engine doesn't fail — it **downgrades** data
batches from GPU → HOST (→ disk), freeing GPU bytes so a stalled reservation can succeed.
"Downgrade" is cuCascade's word for *spilling*. One `downgrade_executor` exists **per
memory space** (e.g. one for GPU:0) and is responsible for downgrading data *out of* that
space.

The whole design is **request-driven and predicate-bounded**: a caller says "free memory
until *this* becomes true," the executor converts batches one at a time, re-checking the
predicate after each, and stops the moment it's satisfied — so it frees *just enough*, not
everything.

## Where this sits

This is another executor in the engine (alongside the per-GPU
[`../pipeline/gpu_pipeline_executor.md`](../pipeline/gpu_pipeline_executor.md) and the
[`task_scheduler`](../pipeline/task_scheduler.md); the
[`../op/scan/duckdb_scan_executor.md`](../op/scan/duckdb_scan_executor.md) is vestigial
post-`#871` and no longer built)—but note it does **not** inherit
`itask_executor`; it's a standalone class with its own thread(s) + pool + request queue.

The critical interaction: when `gpu_pipeline_executor`'s `make_reservation()` comes up
short, it issues **one** `request_downgrade(predicate)` where the predicate is "did
`make_reservation_or_null(bytes_needed)` now succeed?" The downgrade executor frees until
that flips true, the reservation goes through, and the task runs. That's the loop that
turns "GPU full" into "spill a little, continue" instead of OOM. It also reads the
**`task_scheduler`'s pipeline task queue** as a second candidate source (below).

## Two threads + a pool

```
request_downgrade(predicate) / request_free_memory(bytes)
   └─ push downgrade_request onto _request_queue → return std::future<size_t>

processing_loop()                                   :142   (one thread, requests handled SEQUENTIALLY)
└─ for each request, fetch candidates in TIERED order, lazily:
   ├─ Tier 1: data repositories                      :210
   │     convertible_data_batch_provider(repo).get_all_convertible(space, back_to_front)
   ├─ Tier 2: the pipeline task queue                :273   (if wired)
   │     convertible_gpu_pipeline_task_provider(queue).get_next_convertible(...)
   └─ dispatch each candidate to _pool → convertible_data::convert()  (GPU→HOST)
         after each: re-eval predicate; if true, stop dispatching (in-flight finish naturally)

monitor_loop()                                      :372   (optional second thread)
└─ every monitor_period_ms: if should_downgrade_memory(): enqueue a fire-and-forget
   monitor request (proactive spilling before a task even asks)
```

Requests are processed **sequentially** on purpose (cpp comment): concurrent requests
would fight over the same batches. Within a request, the *conversions* run concurrently on
the `bounded_thread_pool`.

## Candidate selection — what gets spilled first

Candidates are fetched **lazily** and in a deliberate order so the cheapest-to-give-up data
goes first:
1. **Data repositories** (the port queues between pipelines) — iterated in manager order;
   within a repo, partitions and batches are walked **back-to-front**, filtering for *idle*
   GPU-resident batches in the source space. Idle batches (no active lock) are safe to move.
2. **Pipeline task queue** — `convertible_gpu_pipeline_task_provider` uses `mutable_pop_if`
   to *temporarily* extract a queued task with convertible batches, converts, and returns
   the task to the queue via RAII on every path (so a spilled-but-not-yet-run task keeps its
   place). This is exactly why the scheduler keeps tasks in its queue (downgrade-visible) —
   the property the pull-signal matcher preserves
   ([`../pipeline/task_scheduler.md`](../pipeline/task_scheduler.md)).

Each candidate goes through `convertible_data::convert()` (cuCascade), which does state
locking, the tier conversion, and failure rollback atomically.

## Methods (hpp)

| Member (signature) | Line | Role |
|---|---|---|
| `downgrade_executor(config, repo_mgr, space_id, memory_space, reservation_mgr, task_queue)` | hpp 83 | bound to one space; optional task-queue for Tier-2 fallback. |
| `std::future<size_t> request_downgrade(predicate)` | hpp 149 | the general entry: free until predicate true. |
| `std::future<size_t> request_free_memory(bytes)` / `..._and_wait(bytes)` | hpp 117 / 127 | convenience predicate = "freed ≥ bytes." |
| `void set_pipeline_task_queue(queue)` | hpp 137 | deferred wiring of the Tier-2 source (must precede `start()`). |
| `void start() / stop() / drain()` | hpp 99–101 | thread lifecycle (drain restarts the processing thread for the next query). |
| `void processing_loop()` / `monitor_loop()` | cpp 142 / 372 | the two loops above (private). |

## Types fundamental to *this* file

- **`downgrade_request`** *(this file; hpp 51)* — a `predicate` + a `promise<size_t>` +
  atomics for bytes/batches freed + an `is_monitor_request` flag. **Think:** "free memory
  until this predicate is happy; here's a future for how much you freed."
- **`convertible_data` / `convertible_data_batch_provider` /
  `convertible_gpu_pipeline_task_provider`** *(Sirius `data/`)* — the abstraction that finds
  spillable batches (in repos / in queued tasks) and performs the tier conversion. **Think:**
  "iterators over 'things I could move off the GPU' + the move itself."
- **`memory_space` / `memory_space_id`** *(cuCascade)* — the space this executor downgrades
  *from*; see [`../memory/sirius_memory_reservation_manager.md`](../memory/sirius_memory_reservation_manager.md).
  **Think:** "the tier I relieve pressure on."
- **`shared_data_repository_manager`** *(cuCascade)* — registry of all port repositories;
  the Tier-1 candidate source. **Think:** "where idle inter-pipeline batches live."
- **`interruptible_mpmc` / `bounded_thread_pool`** *(Sirius exec)* — the request queue +
  conversion worker pool. **Think:** "queue of free-memory asks; pool that does the moving."

## Takeaway

The downgrade executor is the active arm of memory management: a per-space, request-driven
spiller that frees *just enough* GPU memory (predicate-bounded), pulling spill candidates
first from idle repository batches, then from queued tasks, with an optional monitor thread
that spills proactively under pressure. Together with the reservation manager
([`../memory/sirius_memory_reservation_manager.md`](../memory/sirius_memory_reservation_manager.md))
and the GPU executor's OOM-retry, it's what lets Sirius run queries larger than GPU memory
instead of crashing. The other half of "graceful degradation" — falling back to DuckDB CPU
when the GPU path *can't* run a query at all — is
[`../fallback.md`](../fallback.md).
