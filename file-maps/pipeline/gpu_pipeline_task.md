# `pipeline/gpu_pipeline_task.cpp` → Concurrency-Core Map

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), tagged
**study**. Read right after [`gpu_pipeline_executor.md`](gpu_pipeline_executor.md) — it's the
*other half* of that map. Pair with the "Per-task-device contract" and "OOM Handling"
sections of `docs/super-sirius/pipeline-execution.md`; prime memory ideas with
[`../../reference/explainers/gpu-memory-and-spilling.md`](../../reference/explainers/gpu-memory-and-spilling.md).

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> study-note links are relative to this file. Line numbers accurate as of 2026-06-17 —
> re-grep before trusting one; this area moves with the scheduler.

## Where this sits

This is the **task itself** — the thing the scheduler tier passes around. The executor map
([`gpu_pipeline_executor.md`](gpu_pipeline_executor.md)) repeatedly punts here ("its
`execute()` is *not in this file*"); this map is that destination. The unit's lifecycle
threads through all three scheduler components:

- **[`task_creator`](../creator/task_creator.md)** *constructs* a `gpu_pipeline_task` (one
  pipeline + its input batches) and hands it to the scheduler.
- **[`task_scheduler`](task_scheduler.md)** *routes* it — its pull-matcher reads this task's
  **`get_preferred_device_id()`** to pin it to the right GPU.
- **[`gpu_pipeline_executor`](gpu_pipeline_executor.md)** *runs* it — sizes a reservation
  from this task's **`get_estimated_reservation_size_info()`**, dispatches **`execute(stream)`**
  on a worker thread, and on OOM calls this task's **`create_rescheduled_task()`** to resume.
- the operators ([`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md)) are
  what `execute()` actually drives — `op.execute()` per operator, then the sink's `sink()`.

So: **creator builds it · scheduler places it · executor dispatches it · *this* runs the
operators · operators compute.** `gpu_pipeline_task : sirius_pipeline_itask`, and the header
notes it may be *further derived* (build/aggregation task subtypes) — which is why
`create_rescheduled_task()` is `virtual`.

## The key idea first: one resumable run of one pipeline

A `gpu_pipeline_task` is **one execution of one [`sirius_pipeline`](sirius_meta_pipeline.md)
over one slice of input**, and — crucially — it is **resumable**. Its state splits in two
(the cuCascade global/local task-state pattern):

- **global state** (`gpu_pipeline_task_global_state`, an alias of
  `sirius_pipeline_task_global_state`) — shared by every task of the pipeline: the
  `sirius_pipeline*` and the **`memory_history`** (the learned peak-memory feedback used for
  sizing).
- **local state** (`gpu_pipeline_task_local_state`, hpp 57) — this run only: `_input_data`
  (the input batches), **`_start_operator_index`** (resume point — `0` = fresh), the held
  `reservation`, `_preferred_device_id`, and `retry_count` / `original_task_id` for OOM.

`execute()` orchestrates the run; on OOM it doesn't crash — it throws an
`oom_reschedule_exception` carrying the partial output **and the operator index to resume
from**, which the executor turns into a fresh task via `create_rescheduled_task()`. That's the
whole robustness story in one object.

```
execute(stream)                                  :362   ◀─ entry: called by the executor's worker lambda
├─ reservation = local_state.release_reservation() :383   (executor reserved it; take ownership)
│     └─ allocator->attach_reservation_to_tracker  :389   defragmenter_oom_policy; Cleanup resets on exit :391
├─ input_data->prepare_for_processing(mem_space, stream) :412   ══ PER-TASK-DEVICE CONTRACT
│     └─ on rmm::out_of_memory ─▶ throw oom_reschedule_exception(input, 0, …) :424   (resume at op 0)
│     (after this every input batch is locked read-only on the reservation's device)
├─ compute_task(stream)                           :452   ── run the operators ──▶ (below)
├─ memory_history.record({input_basis, peak, output}) :473   feed the estimator for next time
└─ publish_output(output, stream)                 :487   ─▶ sink->sink() pushes into downstream ports

compute_task(stream)                              :239   run operators[_start_operator_index … N)
└─ for i = start … N:                             :264
   ├─ run_one_operator(op, input, stream, …)      :274   ─▶ op.execute() + stream.sync() + type-check
   └─ on rmm::out_of_memory at op i ─▶ throw oom_reschedule_exception(partial, i, …) :327  (resume at op i)

▲ exit: returns final operator_data to execute(); locks release when the input data is destroyed
```

## `execute()` — the per-task-device contract (cpp 362)

The single most important thing this file does. Before any operator runs, **every input batch
is locked-or-converted onto the task's reservation device** via
`local_state._input_data->prepare_for_processing(requested_memory_space, stream)` (cpp 412).
That is the doc's *per-task-device colocation contract*, and it's why an operator's `execute()`
can safely assume `batches[0]->get_memory_space()` is the device it's running on (the caveat
you saw in the operator maps). After `prepare_for_processing` the input batches are held
**read-only** (cpp 444–446); those shared locks release when the input
`pipelineable_operator_data` is destroyed after the first operator consumes it.

Mechanics worth noting:
- the reservation the executor made is **taken over** here (`release_reservation`, cpp 383) and
  attached to a `reservation_aware_resource_adaptor` with a **`defragmenter_oom_policy`**
  (cpp 389–390) — so cuDF allocations during this task draw from the reservation and can
  trigger defrag/spill rather than hard-failing; an `absl::Cleanup` resets the stream
  reservation on exit (cpp 391).
- if `prepare_for_processing` itself OOMs, it throws `oom_reschedule_exception` with resume
  index **0** (cpp 424) — the whole task retries from the top.

## `compute_task()` + `run_one_operator()` — the operator loop (cpp 239 / 150)

`compute_task()` walks `pipeline->get_operators()` from `_start_operator_index` to the end
(cpp 264), threading each operator's output into the next via the anonymous-namespace helper
**`run_one_operator()`** (cpp 150): it calls `op.execute(input, stream)`, **synchronizes the
stream** (cpp 166), records per-operator telemetry/timing, and runs
`validate_operator_output_types()` (cpp 44) — a debug guard that warns if an operator's output
column count/types don't match its declared `get_types()`. If any operator throws
`rmm::out_of_memory`, `compute_task` records the failure into `memory_history` and re-throws as
`oom_reschedule_exception(partial_output, i, …)` (cpp 327) — **resume index = the operator that
OOM'd**, so the retry recomputes from there rather than from scratch.

## `publish_output()` — hand results to the sink (cpp 337)

Calls the pipeline's **sink** operator's `sink(output_data, stream)` (cpp 348) — this is the
push-execution handoff that deposits the task's output into downstream ports/repositories (see
[push vs. pull](../../reference/explainers/push-vs-pull-execution.md)). Throws if the pipeline
has no sink.

## Reservation sizing + the memory-history feedback loop (cpp 507)

`get_estimated_reservation_size_info()` is **what the executor reserves against** (it's read at
`gpu_pipeline_executor` cpp 130/150). It returns `peak_memory_estimate + bytes_to_materialize_input`:
1. compute `input_basis` and `bytes_to_materialize_input` from the local state;
2. ask `memory_history.estimate_peak_memory(input_basis)` (cpp 513) — if this pipeline has run
   on similar input before, use the **learned** peak;
3. **no history** → take the max of each operator's `no_history_peak_memory_estimate(stats)`
   (cpp 535), falling back to `2× input_basis` if every operator is a pass-through (cpp 539).

The loop closes inside `execute()`: after a successful run it records
`{input_basis, peak_bytes, output_bytes}` into `memory_history` (cpp 473), so the *next* task
on this pipeline sizes its reservation from observed reality. This is why an early query can
over- or under-reserve and later ones settle — a small but elegant adaptive-sizing detail.

## OOM resume — `create_rescheduled_task()` (cpp 558)

A `virtual` factory the executor calls (at `gpu_pipeline_executor` cpp 355) after catching
`oom_reschedule_exception`. It builds a **new** `gpu_pipeline_task` reusing the same
`_data_repos` and **shared** global state (so it keeps the same pipeline + memory_history),
with a fresh local state that carries the partial data, the resume index, and an incremented
`retry_count`. Being virtual lets derived task subtypes preserve their dynamic type across a
retry.

## Lifecycle — subscribe/unsubscribe brackets pipeline task-counting

The constructor (cpp 185) `subscribe()`s to each input `data_batch` (ref-counting that keeps
the batch's memory alive while the task references it) and calls **`pipeline->mark_task_created()`**
(cpp 208). The destructor (cpp 212) `unsubscribe()`s and calls **`pipeline->mark_task_completed()`**
(cpp 231). So a task's lifetime brackets the pipeline's outstanding-task count — the same
counter the executor checks via `is_pipeline_finished()` to decide query completion. (See the
created/!completed race the creator closes by calling `mark_task_created` *before* popping —
[`task_creator.md`](../creator/task_creator.md).)

## Methods

| Member (signature) | Line | Role |
|---|---|---|
| `gpu_pipeline_task(task_id, data_repos, local_state, global_state)` | cpp 185 | subscribes to input batches; `pipeline->mark_task_created()`. |
| `~gpu_pipeline_task()` | cpp 212 | unsubscribes; `pipeline->mark_task_completed()`. |
| `void execute(cuda_stream_view)` *(override)* | cpp 362 | the run: take reservation → `prepare_for_processing` (device contract) → `compute_task` → record history → `publish_output`. |
| `operator_data compute_task(cuda_stream_view)` *(override)* | cpp 239 | run operators `[_start_operator_index … N)`; throw `oom_reschedule_exception(partial, i)` on OOM. |
| `void publish_output(operator_data&, cuda_stream_view)` *(override)* | cpp 337 | call the sink's `sink()`. |
| `reservation_size_info get_estimated_reservation_size_info()` *(override)* | cpp 507 | the bytes the executor reserves; uses `memory_history` or per-op estimates. |
| `create_rescheduled_task(task_id, local_state)` *(virtual)* | cpp 558 | OOM-resume factory; same repos + shared global state, new resume index. |
| `get_output_consumers()` *(override)* | cpp 546 | delegates to `pipeline->get_output_consumers()` (what the executor schedules next). |
| `optional<int> get_preferred_device_id()` | hpp 161 | local override → global default; read by the scheduler's matcher. |
| `size_t get_input_size()` | cpp 493 | sum of read-only input batch bytes. |
| `const sirius_pipeline* get_pipeline()` | cpp 234 | the pipeline from global state. |
| *(anon)* `run_one_operator(...)` | cpp 150 | one operator: `op.execute()` + `stream.sync()` + timing + type-check. |
| *(anon)* `validate_operator_output_types(...)` | cpp 44 | debug guard: output columns/types vs `op.get_types()`. |

## Types fundamental to *this* file

- **`gpu_pipeline_task_local_state`** *(Sirius; hpp 57)* — this run's state: `_input_data`,
  `_start_operator_index` (resume), `retry_count`, `_preferred_device_id`. **Think:** "the
  resumable bookmark for one task attempt."
- **`gpu_pipeline_task_global_state`** *(alias of `sirius_pipeline_task_global_state`)* —
  shared per pipeline: the `sirius_pipeline*` + `memory_history`. **Think:** "what every
  attempt of this pipeline shares, including its learned memory profile."
- **`oom_reschedule_exception`** *(Sirius; `pipeline/oom_reschedule_exception.hpp`)* — partial
  output + resume index. **Think:** "not a crash — 'retry me from operator k after you free
  memory.'" Caught by [`gpu_pipeline_executor`](gpu_pipeline_executor.md).
- **`operator_data` / `pipelineable_operator_data`** *(Sirius op)* — the batch-carrying value
  passed operator→operator; the pipelineable subtype exposes read-only locked batches.
  **Think:** "the data flowing down the pipeline, with its residency locks."
- **`reservation_aware_resource_adaptor` / `defragmenter_oom_policy`** *(cuCascade / Sirius
  memory)* — make cuDF allocations draw from the task's reservation and defrag/spill on
  pressure. **Think:** "the allocator that turns OOM into spill, then into a resumable retry."
  Week 6; see [gpu-memory-and-spilling](../../reference/explainers/gpu-memory-and-spilling.md).
- **`memory_history` / `reservation_size_info`** *(Sirius)* — the adaptive sizing feedback.
  **Think:** "learn each pipeline's real peak so the next reservation is right-sized."

## Takeaway

`gpu_pipeline_task` is the **resumable unit of GPU work**: it locks its inputs onto one device
(the per-task-device contract), runs the pipeline's operators in order, pushes results to the
sink, and — on OOM — throws a *resume-from-operator-k* exception the executor reschedules. It
also closes the reservation loop (estimate → run → record actuals → better estimate) and
brackets the pipeline's task count via subscribe/`mark_task_created` … unsubscribe/`mark_task_completed`.
With this, the scheduler tier is complete: [creator](../creator/task_creator.md) (what) →
[scheduler](task_scheduler.md) (where) → [executor](gpu_pipeline_executor.md) (dispatch) →
**task** (run) → [operators](../op/sirius_physical_operator.md) (compute). Next, see the
operators it drives.
