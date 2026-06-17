# `pipeline/task_scheduler.cpp` → Concurrency-Core Map

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md) ("Pipeline
execution & task creation"), tagged **study** — the deepest tier. Read with
`docs/super-sirius/pipeline-execution.md` open (the "Pipeline Executor" + "Management
Event Loop" sections), and prime with
[`../../reference/explainers/morsel-driven-parallelism.md`](../../reference/explainers/morsel-driven-parallelism.md)
(the design philosophy) and
[`../../reference/explainers/cuda-streams-and-async.md`](../../reference/explainers/cuda-streams-and-async.md).

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> study-note links are relative to this file. Line numbers accurate as of 2026-06-15 —
> **this file changes often**, so re-grep the function name before trusting a line.

> **All line numbers below refreshed for pull `d9172de6` (2026-06-15)** and the container
> types corrected (see "State worth knowing"). The pull-signal matcher + RR-counter drift
> still hold. Full diff: [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md).

> ⚠️ **Doc-vs-code drift — read this first.** The doc describes a Phase-14 *round-robin
> counter* (`_no_pref_rr_counter`) that distributes preference-less tasks across GPUs.
> The **code has since moved to a pull-signal model** (`management_eventloop`, cpp 276) —
> GPUs *announce* readiness and the scheduler matches a task to a waiting device. The RR
> counter field survives only as a deprecated testing shim (hpp 226–229 say so
> explicitly). Trust the **code**; treat the doc's RR section as historical. The
> *per-task-device colocation contract* in the doc (every input batch arrives on the
> task's reservation device before `execute()`) is still live and important.

## Where this sits

This is the **top of the scheduler tier** — the box labelled "SCHEDULER" at the bottom of
the [orchestration spine](../README.md). It's handed control by `sirius_engine`
([`../sirius_engine.md`](../sirius_engine.md), Step 5): the engine calls `start_query()`
and blocks on the returned `std::future<void>`. From here down it's all concurrency, no
SQL semantics.

`task_scheduler` **owns the sub-executors** and routes work between them:
- one **`gpu_pipeline_executor` per GPU** ([`gpu_pipeline_executor.md`](gpu_pipeline_executor.md))
  — runs pipeline tasks on a device.
- one **`duckdb_scan_executor`** ([`../op/scan/duckdb_scan_executor.md`](../op/scan/duckdb_scan_executor.md))
  — runs scan tasks (host I/O).
- it holds a `task_creator*` ([`../creator/task_creator.md`](../creator/task_creator.md)),
  the component that *decides which operator becomes the next task*.
- a `completion_handler` (promise/future) is how "query done" travels back to the engine.

Mental split: **`task_creator` decides *what* to run; `task_scheduler` decides *where*
(which GPU) and *when* (which device is free); `gpu_pipeline_executor` actually *runs* it.**

## The key idea first: a pull-signal matcher

The whole file revolves around one thread running `management_eventloop()` (cpp 276). It
is **not** a push loop ("here's a task, find a GPU"). It's a **pull/matcher** loop:

1. GPUs that have a free worker send a `device_ready` event → recorded in `_ready_devices`.
2. `schedule()` (cpp 94) pushes a new task into `_task_queue` and sends `task_available`.
3. On *either* event, the matcher walks each ready device and pops the first task whose
   `preferred_device_id` matches it (or that has *no* preference) — `pop_if` on the queue.

The crucial property (called out in the cpp comment at 288–290): **tasks stay in the
queue until a matching device is free.** That keeps them *downgrade-visible* (the memory
system can spill a queued task's data under pressure) — a property an earlier push model
(PR #732) lost and this loop deliberately restores.

```
management_eventloop()                                    :276
└─ loop while running:
   ├─ evt = _task_request_channel.get()                   :293   block for next event
   │     ├ device_ready  → _ready_devices.emplace_back(dev):298   (vector, not set)
   │     └ task_available→ (just re-run matcher)
   ├─ drain any further queued events (batch the burst)   :303
   └─ matcher: for each ready device:                     :318
        ├─ pop_if task with preferred_device_id == dev    :321   (exact-match first)
        ├─ else pop_if task with NO preference            :333   (or stale pref → treat as none)
        └─ if found ─▶ _gpu_executors.at(dev)->schedule(task):366  ─▶ gpu_pipeline_executor
                       erase dev from _ready_devices       :367
```

Why preference matters: a task may reference GPU-resident data (`cache=table_gpu`,
already-partitioned batches) valid **only** on one device; binding it there is a
correctness requirement, not a perf tweak. Preference-less source tasks (scans) go to
whichever device asks first — that's how multi-GPU load-balances now.

## When does this loop start? (it's already running)

A common confusion: `management_eventloop` is **not** started by `start_query()` or anything
in the query path. It's spawned **once, at context construction** — `SiriusContext::initialize()`
(`sirius_context.cpp`) calls `task_scheduler_->start()` (cpp 602), and `start()` does
`std::thread(&task_scheduler::management_eventloop, this)` (cpp 117). The same `initialize()`
also starts a whole set of **standing threads** that outlive any single query and sit *blocked*
waiting for work:

| Thread (loop) | Spawned at | Blocks on, until… |
|---|---|---|
| `management_eventloop` (this file) | `task_scheduler::start()` cpp 117 | `_task_request_channel.get()` — a `device_ready` / `task_available` event |
| `gpu_pipeline_executor::manager_loop` — **one per GPU** | each `gpu_exec->start()` (from `task_scheduler::start()`, cpp 114) | sends `device_ready`, then `_task_queue.pop()` — a task |
| `task_creator::manager_loop` | `task_creator_->start_thread_pool()` (`sirius_context.cpp:600`) | `_task_creation_queue.pop()` — a `schedule()` request |
| scan executor / downgrade executor(s) | `scan_manager_->start()` / `executor->start()` (cpp 601 / 598) | their own queues |

So by the time a query runs, the matcher thread is alive and parked. `start_query()` doesn't
*enter* it — it feeds the `task_creator`, whose output eventually `schedule()`s a task here,
which sends `task_available` on the channel and **wakes the already-running loop**. The full
beginner-friendly walkthrough (what each thread waits on, when it wakes, what `start_query`
hands back) is in
[query startup & the thread model](../../reference/explainers/query-startup-and-threads.md).

## Query lifecycle — the methods the engine drives

| Method (signature) | Line | Role |
|---|---|---|
| `task_scheduler(gpu_cfg, scan_cfg, mem_mgr, topology, downgrade)` | 40 | builds one `gpu_pipeline_executor` per GPU + the `duckdb_scan_executor`. |
| `void start()` | 110 | **one-time, at context init** (not per query): start each GPU sub-executor, then `_management_thread = std::thread(&task_scheduler::management_eventloop, this)` (cpp 117). After this the matcher thread is alive and parked on the channel. |
| `void prepare_for_query(query)` | 147 | **the per-query setup** (called from `SiriusContext::create_query`, *before* `start_query`): drain leftover tasks from the prior query (150), store `_query` (155), **create the `completion_handler` (157) and hand it to each GPU executor (161)**, reset the (deprecated) RR counter (168). |
| `std::future<void> start_query()` | 171 | **the trigger — tiny.** Just `_task_creator->schedule(scans.front())` (176, drop the first scan in) and `return _completion_handler->get_awaitable()` (178) — the future the engine blocks on. *All* setup already happened in `prepare_for_query`; this only ignites the cascade. |
| `void schedule(itask)` | 94 | push a pipeline task into `_task_queue`, signal `task_available`. Called by the GPU executor (downstream consumers) and the creator. |
| `void terminate_query(exception_ptr)` | 181 | report error to the completion handler (unblocks the engine with an exception). |
| `void drain_after_error()` | 187 | the careful multi-stage shutdown (stop creator → drain queue → drain GPU execs → drain scan exec → restart creator) so QueryEnd can destroy repositories without a use-after-free. |
| `void wait_for_completion()` | 235 | ensure no tasks in flight before plan teardown. |
| `void management_eventloop()` | 276 | the matcher above (private). **Entry point:** runs on `_management_thread`, spawned in `start()` (`std::thread(&task_scheduler::management_eventloop, this)`, cpp 117). `start()` is called *once* at context setup — `SiriusContext::initialize()` → `task_scheduler_->start()` (`src/sirius_context.cpp:602`), **not** per query — so the loop is long-lived (context lifetime), not per-`start_query()`. Exits when `stop()` closes `_task_request_channel` (cpp 129) and joins the thread (cpp 132). |

## State worth knowing (hpp)

- `_gpu_executors` — **`std::unordered_map<int, …>`** (hpp 225). *(Heads-up: the in-source
  comment just above the declaration still says "std::map (not unordered_map) … deterministic"
  — that comment is **stale** as of this pull; the type is `unordered_map`. Under the
  pull-signal model determinism comes from device_ready arrival order + binding device
  preference, not from map iteration order, so the revert is benign. Flagging because the
  comment contradicts the code.)*
- `_ready_devices` (**`std::vector<int>`**, hpp 220) — devices with a parked worker; mutated
  **only** by the management thread, so no lock needed (confinement over locking). *(Was a
  `std::set<int>` in our earlier read; now a vector — the matcher iterates + `erase`s it.)*
- `_task_queue` (`inspectable_mpsc<itask>`, hpp 207) — the pending pipeline tasks; supports
  `pop_if` for the preference matcher.
- `_task_request_channel` (hpp 208) — the event bus carrying `device_ready` / `task_available`.
- `_no_pref_rr_counter` (hpp 229) — **deprecated**, kept only for
  `set_no_pref_rr_counter_for_testing` (stress tests that perturb scheduling order).
- `_task_queue_telemetry` (hpp 234) — new this pull (Quent task instrumentation, `#809`);
  not part of the scheduling logic.

## Types fundamental to *this* file

- **`task_request` / `task_request_kind`** *(Sirius)* — the event objects on the channel
  (`device_ready`, `task_available`, plus `is_scan`). **Think:** "a GPU saying 'I'm free'
  or `schedule()` saying 'new work.'"
- **`itask` / `gpu_pipeline_task`** *(Sirius)* — the unit of schedulable work; the matcher
  dynamic-casts to `gpu_pipeline_task` to read its `preferred_device_id`. **Think:** "one
  pipeline run, possibly pinned to a device." Mapped in
  [`gpu_pipeline_task.md`](gpu_pipeline_task.md); dispatched by
  [`gpu_pipeline_executor.md`](gpu_pipeline_executor.md).
- **`completion_handler`** *(Sirius; promise/future)* — first `mark_completed()` /
  `report_error()` wins (CAS); `get_awaitable()` is the future the engine waits on.
  **Think:** "the single wire that says the query is done (or failed)."
- **`memory_space` / `sirius_memory_reservation_manager`** *(Sirius/cuCascade)* — per-GPU
  memory the executors reserve against; Week 6 territory. **Think:** "the GPU's allocator
  the scheduler hands to each task."
- **`exec::inspectable_mpsc` / `exec::channel` / `exec::publisher`** *(Sirius exec)* — the
  lock-aware queue + event primitives. **Think:** "the plumbing; don't reimplement them,
  just trust the matcher uses them correctly."

## Takeaway

`task_scheduler` is the *router*: a single management thread matches ready GPUs to
queued tasks (honoring device preference), owns the per-GPU and scan sub-executors, and
carries query completion back to the engine via a future. The big honesty note is the
**pull-signal vs. doc's RR-counter drift** — verify against code. Next, see what a GPU
executor actually does with a dispatched task (memory reservation, OOM retry, downstream
scheduling) in [`gpu_pipeline_executor.md`](gpu_pipeline_executor.md), and how tasks get
*created* in [`../creator/task_creator.md`](../creator/task_creator.md).
