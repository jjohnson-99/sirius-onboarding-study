# `memory/sirius_memory_reservation_manager.cpp` → Memory-Tier Map

Companion for **Week 6** of [`onboarding-path.md`](../../onboarding-path.md) ("Memory &
fallback"), tagged **read closely** — reservations and tiering are easy to get subtly
wrong. Read with `docs/super-sirius/memory-management.md`; prime with
[`../../reference/explainers/gpu-memory-and-spilling.md`](../../reference/explainers/gpu-memory-and-spilling.md),
[`../../reference/explainers/cucascade.md`](../../reference/explainers/cucascade.md), and
[`../../reference/explainers/rmm.md`](../../reference/explainers/rmm.md).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — re-grep.

> **Calibration up front: this "read closely" file is *tiny*.** The header is 43 lines and
> the `.cpp` is 72 — because `sirius_memory_reservation_manager` is a **thin subclass** of
> cuCascade's `memory_reservation_manager`; the real reservation/tiering/spilling machinery
> lives in **cuCascade**, not here. So "read closely" means *read closely the concepts in
> `memory-management.md` + the cuCascade explainer*, and read this file to see exactly
> where Sirius plugs in. Don't expect to find the allocator here.

## Where this sits

Memory is the substrate under everything you read in Weeks 4–5. The reservation manager is
**constructed once** and handed to the `task_scheduler`
([`../pipeline/task_scheduler.md`](../pipeline/task_scheduler.md)); from it, each
`gpu_pipeline_executor` ([`../pipeline/gpu_pipeline_executor.md`](../pipeline/gpu_pipeline_executor.md))
gets a `memory_space` to `make_reservation()` against before running a task. When a
reservation can't be satisfied, the **downgrade executor**
([`../downgrade/downgrade_executor.md`](../downgrade/downgrade_executor.md)) spills data to
a lower tier and the task retries. So this file is the *root* of the memory story; the
downgrade map is its active half.

## The key idea first: three tiers, reserve-before-run

cuCascade manages memory across **three tiers**, and Sirius reserves capacity *up front*
so a task can't blow past the GPU mid-run:

```
GPU  (Tier 0)  fast, small   (~24 GB)   — pipeline-task computation
HOST (Tier 1)  medium, big   (>100 GB)  — pinned pools per NUMA; caching + transfer staging
DISK (Tier 2)  slow, huge    (~1 TB)    — spill files; last resort
```

Each tier has fractions (`reservation_limit` 0.9, `downgrade_trigger`, `downgrade_stop`)
that decide when spilling kicks in — the *policy* numbers in `memory-management.md`'s
table. Two reservation facts to carry:
- `make_reservation(bytes)` is taken in the GPU executor's manager loop **before**
  `execute()`; on shortfall it triggers downgrade then retries; on exhaustion mid-run an
  allocation throws `oom_reschedule_exception` (the resumable-retry you saw in the GPU
  executor map).
- the **`reservation_aware_resource_adaptor`** (cuCascade) wraps the RMM resource so every
  allocation is checked against the reservation — that's what makes per-task memory
  *predictable* instead of best-effort.

## What *this file* actually does (cpp 27–72)

Exactly one job, and it's a lifetime/correctness guard worth understanding:

| Member (signature) | Line | Role |
|---|---|---|
| `sirius_memory_reservation_manager(configs)` | cpp 27 | calls the cuCascade base ctor (which builds the tiers from `memory_space_config`), then for **each GPU space** saves the *current* cuDF device resource (`prev_device_mrs_`) and **points cuDF at the cuCascade allocator** (`cudf::set_current_device_resource_ref`). Throws if no GPU space configured. |
| `~sirius_memory_reservation_manager()` | cpp 39 | for each GPU: `cudaDeviceSynchronize()` then **restore** the saved cuDF resource. |

Why both halves matter (the subtle correctness points the comments call out):
- **Saving/restoring** the cuDF device resource (rather than resetting it) prevents a
  dangling/null resource that would crash later allocations once Sirius's custom allocators
  are torn down — important because tests construct/destruct this repeatedly.
- The **`cudaDeviceSynchronize()` at teardown** drains pending stream-ordered frees
  (`cudaFreeAsync`) against the pool about to be destroyed; without it, an un-synced async
  deallocation can corrupt the driver's per-device pool list and crash the *next* manager
  built on the same device. A concrete async-lifetime footgun (see
  [`../../reference/explainers/cuda-streams-and-async.md`](../../reference/explainers/cuda-streams-and-async.md)).

So the file is small but load-bearing: it **wires cuDF to use cuCascade's tiered allocator
on every GPU**, and unwinds that cleanly.

## The rest of `src/memory/` — `defragmenter_oom_policy` (read alongside)

`src/memory/defragmenter_oom_policy.cpp` implements `cucascade::memory::oom_handling_policy`
— the hook cuCascade calls **on an allocation failure before giving up**:
1. query CUDA pool fragmentation via `cudaMemPoolGetAttribute()` (reserved vs. used);
2. if `reserved > used + 10× requested`, the pool is fragmented;
3. `cudaMemPoolTrimTo()` to return free blocks to the driver;
4. retry; if still failing, rethrow.
This is the "maybe it's not really out of memory, just fragmented" escape valve before
downgrade/OOM-retry. Small (103 lines), worth reading for the CUDA-pool mental model.

## Reservation *sizing* — `pipeline_memory_history` (concept pointer)

How big a reservation does a task ask for? `src/include/pipeline/pipeline_memory_history.hpp`
keeps a per-pipeline ring buffer of `{estimated, peak, output}` records and computes a
weighted average peak/estimated ratio (`estimate_peak_memory`). On OOM it keeps the
**higher** peak so each retry reserves more. You don't need to map it, but know it's why
reservations adapt over a query rather than guessing once.

## Types fundamental to *this* file

- **`cucascade::memory::memory_reservation_manager`** *(cuCascade base)* — the real tiered
  manager; this class subclasses it. **Think:** "the actual memory brain; Sirius just
  configures it and bridges cuDF."
- **`memory_space` / `memory_space_config`** *(cuCascade)* — one tier on one device/NUMA/disk;
  the thing executors `make_reservation()` against. **Think:** "a pool with a budget and a
  tier."
- **`memory_reservation`** *(cuCascade)* — a claimed slice of a space, attached to a task.
  **Think:** "this task's pre-approved memory budget."
- **`rmm::device_async_resource_ref` / cuDF device resource** *(RMM/cuDF)* — the allocator
  handle this file saves/restores so cuDF allocates from cuCascade. **Think:** "the wire
  that makes `cudf::*` calls use Sirius's tiered pool." See
  [`../../reference/explainers/rmm.md`](../../reference/explainers/rmm.md).
- **`defragmenter_oom_policy`** *(this dir)* — pool-trim-then-retry on alloc failure.
  **Think:** "defrag before you despair."

## Takeaway

The reservation manager itself is a thin, careful bridge: it configures cuCascade's
three-tier allocator and points cuDF at it on every GPU, with teardown discipline to avoid
async-free corruption. The *behavior* you care about — reserve-before-run, fraction-driven
downgrade triggers, fragmentation trimming, adaptive reservation sizing — lives in
cuCascade + the policy/history helpers. The active spilling half is next:
[`../downgrade/downgrade_executor.md`](../downgrade/downgrade_executor.md).
