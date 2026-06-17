# Query startup & the thread model (how `start_query` launches a query)

If you read `sirius_engine::execute()` and then `task_scheduler::start_query()`, you hit a
puzzle: `start_query()` is **four lines**, yet calling it sets a whole GPU query in motion.
Where's the work? The answer is that Sirius doesn't *start* an engine when a query arrives — it
**triggers** an engine that has been running, idle, since the database was set up. This note
unpacks that, assuming **no prior threading background**.

> **Prime around:** Week 4 (the scheduler tier). Read after the scheduler maps
> ([`task_scheduler`](../../file-maps/pipeline/task_scheduler.md),
> [`gpu_pipeline_executor`](../../file-maps/pipeline/gpu_pipeline_executor.md),
> [`gpu_pipeline_task`](../../file-maps/pipeline/gpu_pipeline_task.md),
> [`task_creator`](../../file-maps/creator/task_creator.md)) and
> [`sirius_engine`](../../file-maps/sirius_engine.md). Conceptual cousins:
> [push vs. pull](push-vs-pull-execution.md) and [morsel-driven parallelism](morsel-driven-parallelism.md).

## A threading crash course (just enough)

If you're new to threads, here are the five ideas this note needs. Nothing more.

1. **A thread is an independent line of execution.** Your program normally runs one statement
   after another on a single thread. You can ask the OS for *more* threads, and they run
   **at the same time** (truly in parallel on multiple CPU cores). Each can be running a
   different function.

2. **A thread can *block*.** When a thread has nothing to do, you don't want it spinning in a
   busy loop burning CPU. Instead it calls something that **parks** it — the OS puts the thread
   to sleep until a specific event happens, then wakes it. "Blocked on X" means "asleep until X
   happens." A blocked thread uses ~no CPU. This is the key trick: a thread can sit *ready but
   idle* for a long time at near-zero cost.

3. **A queue is a thread-safe mailbox.** One thread `push`es items in; another `pop`s them out.
   If you `pop` an empty queue, you **block** until someone pushes. This is how one thread hands
   work to another safely. (Sirius's `_task_queue` and `_task_creation_queue` are this.)

4. **A channel is a queue used for *signals/events*.** Same idea, but the items are little
   "something happened" messages rather than work payloads. A thread `get()`s from the channel
   and blocks until an event arrives. (Sirius's `_task_request_channel` carries `device_ready`
   and `task_available` events.)

5. **A promise/future is a one-shot "are we done yet?" wire between two threads.** Thread A
   holds a **future** and calls `future.get()`, which **blocks** until thread B (anywhere)
   fulfills the matching **promise** exactly once. Then `get()` returns and A wakes up. Sirius's
   `completion_handler` is exactly this: `get_awaitable()` hands back the future; a worker
   thread later calls `mark_completed()` to fulfill it.

That's the whole vocabulary: **threads that block on queues/channels/futures, woken by other
threads pushing work or signalling events.** Sirius's execution engine is just a set of such
threads wired together. A system built this way is called **event-driven**: nothing polls in a
loop; each thread sleeps until a specific event, acts, and goes back to sleep.

## The cast: Sirius's standing threads

When the Sirius extension initializes a database context (`SiriusContext::initialize()`, run
**once**, long before any query), it spawns a fixed crew of long-lived threads — call them
*daemon* or *standing* threads. Each runs a loop, and each immediately **blocks**, waiting for
work. They stay alive for the whole context, across many queries.

| Thread (the loop it runs) | How many | Spawned by | Sits blocked on… |
|---|---|---|---|
| **`task_creator::manager_loop`** — decides *what* to run next | 1 | `task_creator_->start_thread_pool()` (`sirius_context.cpp:600`) | `_task_creation_queue.pop()` — a `schedule()` request |
| **`task_scheduler::management_eventloop`** — matches tasks to free GPUs | 1 | `task_scheduler::start()` cpp 117 (via `..._->start()`, `sirius_context.cpp:602`) | `_task_request_channel.get()` — a `device_ready`/`task_available` event |
| **`gpu_pipeline_executor::manager_loop`** — runs pipeline tasks on a GPU | **one per GPU** | each `gpu_exec->start()` (from `task_scheduler::start()`, cpp 114) | first sends `device_ready`, then `_task_queue.pop()` — a task |
| **scan executor / downgrade executor(s)** — host I/O, spilling | a few | `scan_manager_->start()` / `executor->start()` (cpp 601 / 598) | their own queues |

Read that table as: *five-ish threads, all asleep, each waiting for a different doorbell.* This
is why the source is spread across files — each thread's loop lives in its own file
([creator](../../file-maps/creator/task_creator.md),
[scheduler](../../file-maps/pipeline/task_scheduler.md),
[executor](../../file-maps/pipeline/gpu_pipeline_executor.md)), and they only ever talk through
queues and channels.

> One subtlety: each GPU executor's loop, on startup, **announces itself** by sending a
> `device_ready` event *before* it has any task (it reserves a worker slot, sends `device_ready`,
> then blocks on `_task_queue.pop()`). So even with zero queries, the scheduler's matcher has
> already recorded "GPU 0 is free, GPU 1 is free" in `_ready_devices`. It runs its matcher,
> finds no tasks, and goes back to sleep. The devices are *pre-armed*; only a task is missing.

## Before the query: everyone is parked

So the steady state, with the database open and no query running, is:

```
task_creator loop      ──blocked on── _task_creation_queue.pop()        (no requests yet)
management_eventloop   ──blocked on── _task_request_channel.get()       (devices ready, no tasks)
gpu executor 0 loop    ──blocked on── _task_queue.pop()                 (already sent device_ready)
gpu executor 1 loop    ──blocked on── _task_queue.pop()
scan / downgrade loops ──blocked on── their queues
```

Zero CPU burned. The engine is a coiled spring.

## The engine calls `execute()`: setup, then trigger, then wait

Now a query arrives. `sirius_engine::execute()` (Step 5) runs on **the query thread** — the
thread DuckDB is using to pull this query's results (it called down into Sirius). It does three
things, and only the first two touch the scheduler:

```
execute()                                  on THE QUERY THREAD
├─ ctx->create_query(new_scheduled)   :168  ─▶ task_scheduler::prepare_for_query   ── SETUP
│      • store _query                                 (the pipeline tree from Step 4)
│      • _completion_handler = make_unique<…>         (mint a fresh promise/future)
│      • hand completion_handler to every GPU executor
├─ task_scheduler.start_query()       :173  ─▶                                     ── TRIGGER
│      • _task_creator->schedule(scans.front())       (enqueue the FIRST scan)
│      • return _completion_handler->get_awaitable()  (the future)
└─ future.get()                       :175  ── THE QUERY THREAD BLOCKS HERE ──
```

The heavy per-query setup is **`prepare_for_query`** (not `start_query`): it stores the plan,
creates the `completion_handler` (the promise/future the whole query hinges on), and wires that
handler into every GPU executor so they can later signal "done." Only *after* that does
`start_query()` run.

### What `start_query()` actually does, hands back, and causes

It is genuinely tiny — and now you can read it precisely:

- **What it does:** takes the query's scan operators and calls
  `_task_creator->schedule(scans.front())`. That's a single `push` into the **task_creator's**
  `_task_creation_queue`. It does *not* run a scan, touch a GPU, or build a task itself — it
  just drops one "please make a task for this scan" request into a mailbox.
- **What it hands back:** `_completion_handler->get_awaitable()` — the **future**. This is the
  one wire the query thread will wait on. Nobody has fulfilled it yet.
- **What it causes:** that single `push` is the pebble. It wakes the task_creator thread (which
  was blocked on `pop()`), and from there a chain reaction ripples across all the standing
  threads (next section).

Then `execute()` calls **`future.get()`** and the query thread **blocks** — it sleeps until some
*other* thread, much later, fulfills the promise via `completion_handler->mark_completed()`. The
query thread does no query work at all; it's just the thing waiting for the answer.

## The cascade: one pebble, ripples across threads

Here's the chain reaction `start_query()` sets off. Watch *which thread* wakes at each step —
each was blocked, and a `push`/`send` from the previous thread wakes it:

```
[query thread]   start_query(): push "make task for scan" → _task_creation_queue
                       │
                       ▼ wakes
[creator thread] task_creator::manager_loop pop()s the request, builds a scan task,
                 dispatches it to the scan executor   (scans are host I/O)
                       │
                       ▼ wakes
[scan thread]    runs the scan, produces input batches, then schedule()s the scan's
                 downstream consumer(s) back to the task_creator
                       │
                       ▼ wakes
[creator thread] builds a gpu_pipeline_task for the consumer pipeline, calls
                 task_scheduler::schedule(task): push → _task_queue, send task_available → channel
                       │
                       ▼ wakes
[mgmt thread]    management_eventloop get()s task_available, runs the matcher: picks a
                 ready GPU, pops the matching task, calls gpu_executor->schedule(task)  → _task_queue
                       │
                       ▼ wakes
[gpu exec thread] manager_loop pop()s the task, reserves GPU memory, dispatches the task's
                  execute() on a worker → runs the operators (see gpu_pipeline_task.md)
                       │
                       ├─ on success: schedule()s the NEXT pipeline's consumers → loops back up
                       └─ when the final sink is RESULT_COLLECTOR & the pipeline is done:
                              completion_handler->mark_completed()
                       │
                       ▼ fulfills the promise
[query thread]   future.get() returns — the query thread WAKES, execute() finishes,
                 control returns up to fetch the materialized result (Step 9)
```

Every arrow is "thread A pushes/sends → thread B, which was asleep, wakes up." The work hops
from thread to thread; no single thread runs the whole query. Intermediate pipelines just keep
re-triggering the cycle (executor finishes a task → schedules its consumers → creator builds
their tasks → scheduler matches → executor runs them) until the result-collector sink fires the
completion signal.

## So why is `start_query()` so small?

Because **the engine was already built and running.** `start_query()`'s job isn't to construct
or launch anything — the threads exist, the channels exist, the GPUs have announced themselves.
Its job is to **inject the first unit of work** and **hand back the wire to wait on**. The size
of a function is a terrible proxy for what it sets in motion when the system is event-driven: a
4-line trigger can light up five threads and a fleet of GPUs because all that machinery is
standing by, blocked, waiting for exactly that nudge.

This design buys real things:
- **Decoupling:** the query thread, the decider (creator), the router (scheduler), and the
  runners (executors) are separate threads that only share queues — easy to reason about, and
  each can make progress independently.
- **Backpressure:** a GPU only pulls a task when it has a free worker slot (it sends
  `device_ready` first). Work waits in queues where the memory system can still see (and spill)
  it, rather than being shoved onto a busy GPU. (See the "pull-signal matcher" in
  [`task_scheduler`](../../file-maps/pipeline/task_scheduler.md).)
- **Multi-GPU for free:** because tasks are matched to whichever device announces readiness, the
  same trigger scales across N GPU executor threads with no extra orchestration.

## Recap

- Sirius spins up a fixed crew of **standing threads** at `SiriusContext::initialize()` — the
  task_creator, the scheduler's `management_eventloop`, one GPU executor per device, plus
  scan/downgrade executors. They run **once per context**, then sit **blocked**, each waiting on
  a queue or channel. They are *not* per-query and *not* started by `start_query()`.
- `management_eventloop()` is "entered" exactly once, at startup (`task_scheduler::start()`
  spawns its thread, cpp 117); thereafter it's *woken*, not entered.
- The per-query **setup** lives in `prepare_for_query` (mint the `completion_handler`, wire it to
  executors), called from `create_query` *before* `start_query`.
- `start_query()` is just the **trigger**: it `schedule()`s the first scan (one queue push) and
  **returns the future** the query thread blocks on. It hands back a `std::future<void>`; it
  causes a cross-thread cascade; it does no query work itself.
- The query thread then **blocks on `future.get()`** until a GPU worker thread, at the end of the
  pipeline, calls `mark_completed()` — fulfilling the promise and waking the query thread to
  collect results.
