# `pipeline/gpu_pipeline_executor.cpp` → Concurrency-Core Map

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), tagged
**study**. Read with the "GPU Pipeline Executor" + "Manager Loop" + "OOM Handling"
sections of `docs/super-sirius/pipeline-execution.md`, right after
[`task_scheduler.md`](task_scheduler.md).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — re-grep; this area moves.

> **All line numbers below refreshed for pull `d9172de6` (2026-06-15).** The
> **`MAX_OOM_RETRIES = 100` vs. doc's 10 drift still holds** (now cpp 311). Flow + OOM-retry
> logic unchanged. Full diff: [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md).

## Where this sits

**One `gpu_pipeline_executor` exists per GPU device.** The `task_scheduler`
([`task_scheduler.md`](task_scheduler.md)) matches a task to a ready device and calls
*this* executor's `schedule()`. This file is the layer that turns "run this task on this
GPU" into actual work: reserve memory, pick a stream, dispatch to a worker thread, handle
out-of-memory, and schedule whatever comes next.

It inherits from **`itask_executor`** (`src/include/parallel/task_executor.hpp`), the
shared base for all three executor kinds (`gpu_pipeline_executor`, `duckdb_scan_executor`
[`../op/scan/duckdb_scan_executor.md`](../op/scan/duckdb_scan_executor.md), and the
downgrade executor). The base provides the thread pool, queue, `_running` flag, and
`start/stop/schedule/drain_and_wait`; subclasses implement `manager_loop()` and optional
per-thread init. So when you read this, you're reading the **GPU specialization** of a
template the scan/downgrade executors also fill in.

## The key idea first: a reserve → request → pop → dispatch loop

The whole executor is one method, `manager_loop()` (cpp 91), running on a dedicated
thread. The shape (matches the doc's pseudo-loop):

```
manager_loop()                                              :91   ◀─ entry: one per GPU, on its own thread
                                                                  (base itask_executor::start() spawns it)
└─ loop while _running:                                     :98
   ├─ slot = _bounded_pool->reserve()      block until a worker SLOT is free (RAII)     :99
   ├─ _task_request_publisher.send(device_ready)   tell the scheduler "I can take work" :112
   ├─ task = _task_queue.pop()              block until the scheduler hands me a task    :116
   ├─ reservation = _memory_space->make_reservation(bytes)                              :150
   │     └─ short by N & downgrade set ─▶ request_downgrade() to spill, then retry :160 / :209
   ├─ consumers = gpu_task->get_output_consumers()   (captured now, used by the worker) :261
   ├─ stream = _stream_pool.acquire_stream()          one CUDA stream per worker        :263
   └─ _bounded_pool->dispatch(std::move(slot), lambda)                                  :265
            ─── lambda runs on a WORKER thread; the manager thread loops back here ───
         ├─ task->execute(stream)                     ── the pipeline runs here         :273
         │     └─ throws oom_reschedule_exception ─▶ OOM retry path (section below)      :274
         ├─ on success: task.reset()                  releases the reservation          :395
         ├─ check query completion BEFORE scheduling  (don't touch destroyed operators) :401
         ├─ if !complete: task_creator->schedule(consumer) for each                     :416
         └─ if  complete: completion_handler->mark_completed()  → unblocks engine.execute() :422

▲ exit: loop breaks when stop() clears _running, or the pool/channel/queue is interrupted :100/112/116
```

Two layered concurrency limits are worth internalizing: the **`bounded_thread_pool`**
hands out *slots* via `reserve()` (RAII — released when the dispatched lambda finishes),
so the manager never over-subscribes the GPU; and the **`device_ready` send** is the
exact signal the scheduler's pull-matcher
([`task_scheduler.md`](task_scheduler.md#the-key-idea-first-a-pull-signal-matcher)) waits
for. This file is the *other half* of that handshake.

## What `task->execute(stream)` does (the per-task-device contract)

`task` is a `gpu_pipeline_task`. Its `execute()` (in `src/pipeline/gpu_pipeline_task.cpp`,
not this file) is where the doc's **per-task-device colocation contract** lives — before
any operator runs, every input batch is locked-or-converted onto the task's reservation
device (`prepare_for_processing` → `lock_or_prepare_batch`). That's why an operator can
safely read `batches[0]->get_memory_space()` (you saw this caveat in the operator maps).
`compute_task()` then runs every operator's `execute()` in order; `publish_output()` runs
the sink's `sink()` to push results into downstream ports. Read the doc's "Per-task-device
contract under SCHED-RR" section alongside — it's the authoritative four-layer writeup.

## OOM handling — the retry machine (cpp 274–374)

When an operator runs out of GPU memory mid-pipeline it throws
`oom_reschedule_exception` carrying the partial results and the **operator index to
resume from**. The executor catches it and:

1. skip if the completion handler already has an error;
2. increment `retry_count`; **`MAX_OOM_RETRIES = 100`** (cpp 311) — *(note: the doc says
   10; the code says 100 — trust the code)*;
3. transition the partial data back to a schedulable state, build a new task via the
   `create_rescheduled_task()` virtual factory (cpp 355) with `_start_operator_index` set
   so it **resumes mid-pipeline** rather than recomputing;
4. brief backoff, then `this->schedule()` the new task (cpp 374).

Exceeding the cap propagates the error and terminates the query. This is the mechanism
that lets a too-big intermediate get spilled (by downgrade) and retried instead of
crashing the query — a core robustness property of the engine.

## Downstream scheduling (cpp 398–422)

After a task succeeds:
1. get `output_consumers` — the first operator of each parent pipeline;
2. **check query completion *before* scheduling** (ordering matters: prevents scheduling
   tasks that reference already-destroyed operators);
3. if not complete: `task_creator->schedule(consumer)` for each
   ([`../creator/task_creator.md`](../creator/task_creator.md));
4. if the sink is `RESULT_COLLECTOR` and the pipeline is finished:
   `completion_handler->mark_completed()` — the signal that unblocks `engine.execute()`.

## Methods (hpp)

| Member (signature) | Line | Role |
|---|---|---|
| `gpu_pipeline_executor(config, mem_space, task_request_publisher, downgrade_executor)` | hpp 71 | one per GPU; binds its memory space + the publisher to the scheduler. |
| `void manager_loop()` *(override)* | cpp 91 | the reserve→request→pop→dispatch loop above. |
| `AnyInvocable get_per_thread_init()` *(override)* | cpp 62 | pins each worker thread to this GPU device. |
| `void set_task_creator(...)` / `set_completion_handler(...)` | hpp 94 / 111 | wired up by the scheduler at query start. |
| `gpu_pipeline_task* cast_to_gpu_pipeline_task(itask*)` | hpp 126 | typed-cast guard. |

## Types fundamental to *this* file

- **`gpu_pipeline_task`** *(Sirius; `pipeline/gpu_pipeline_task.{hpp,cpp}`)* — the unit
  this executor runs. Holds a `sirius_pipeline` (global state) + input batches, reservation,
  `_start_operator_index`, `retry_count` (local state). **Think:** "one run of one
  pipeline, resumable from an operator index after OOM."
- **`oom_reschedule_exception`** *(Sirius)* — partial results + resume index. **Think:**
  "not a crash — a 'try me again from operator k after you free memory.'"
- **`exec::bounded_thread_pool`** *(Sirius exec)* — slot-based worker pool;
  `reserve()`→`dispatch(slot, fn)` with RAII slot release. **Think:** "the throttle that
  keeps the GPU from being over-subscribed."
- **`exclusive_stream_pool` / `cuda_stream_view`** *(cuCascade/RMM)* — one CUDA stream per
  worker, so concurrent tasks don't serialize on the default stream. **Think:** "each
  worker gets its own GPU lane." See
  [`../../reference/explainers/cuda-streams-and-async.md`](../../reference/explainers/cuda-streams-and-async.md).
- **`memory_space` / reservation** *(cuCascade)* — GPU memory the task reserves before
  running; failure routes to the downgrade executor. Week 6. **Think:** "claim the GPU RAM
  up front; spill something if you can't."

## Takeaway

The per-GPU executor is where scheduling becomes execution: it signals readiness to the
scheduler, reserves memory + a stream, runs the pipeline task on a bounded worker pool,
and — crucially — turns OOM into a *resumable retry* rather than a failure, then schedules
the next pipeline's tasks. Two honesty notes for the friend: the **`MAX_OOM_RETRIES`
doc/code mismatch (10 vs 100)** and the pull-signal `device_ready` handshake that pairs
with the scheduler. Next: who decides *which operator* becomes a task —
[`../creator/task_creator.md`](../creator/task_creator.md).
