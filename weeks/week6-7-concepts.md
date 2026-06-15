# Weeks 6–7 Concepts — Memory, Graceful Degradation, and a Non-Trivial Contribution

The Weeks 6–7 checkpoint in [`onboarding-path.md`](../onboarding-path.md):

> Can make non-trivial contributions and reason about correctness across the concurrency
> and memory layers.

Weeks 4–5 gave you *how work runs concurrently*. Weeks 6–7 add the last layer —
**robustness under memory pressure and graceful degradation** — and then turn you loose to
**ship something non-trivial**. Week 6 is depth on memory + fallback; Week 7 is workflow.

The through-line for Week 6 is a single question:

> **What happens when a GPU pipeline task needs more memory than is free?**

Follow that one question and every Week-6 file falls into place.

## Part 1 — Reserve before you run (memory tiers)

**File map:**
[`../file-maps/memory/sirius_memory_reservation_manager.md`](../file-maps/memory/sirius_memory_reservation_manager.md)
· **doc:** `docs/super-sirius/memory-management.md` · **prime:**
[`../reference/explainers/gpu-memory-and-spilling.md`](../reference/explainers/gpu-memory-and-spilling.md),
[`../reference/explainers/cucascade.md`](../reference/explainers/cucascade.md),
[`../reference/explainers/rmm.md`](../reference/explainers/rmm.md)

cuCascade manages memory across **three tiers** — GPU (fast, small), host-pinned (medium,
big), disk (slow, huge) — each with fractions that decide when to start/stop spilling. The
engine reserves capacity **before** a task runs (in the GPU executor's manager loop), so a
task can't silently blow past the GPU mid-flight; a `reservation_aware_resource_adaptor`
checks every allocation against that reservation.

The honest surprise here: `sirius_memory_reservation_manager` — a "read closely" file — is
*tiny* (43-line header), because the real machinery is **cuCascade's**. Sirius's subclass
just configures the tiers and **points cuDF at cuCascade's allocator on every GPU**, with
careful save/restore + `cudaDeviceSynchronize()` at teardown to avoid async-free
corruption. So "read closely" really means *read the doc + the cuCascade explainer
closely*, and read this file to see the seam. (How big a reservation? `pipeline_memory_history`
learns it from past peaks — adaptive, raised on each OOM retry.)

## Part 2 — When the reservation can't be met: spill, defrag, retry

**File map:**
[`../file-maps/downgrade/downgrade_executor.md`](../file-maps/downgrade/downgrade_executor.md)

This is the heart of "run a query bigger than GPU memory." When `make_reservation()` comes
up short, three mechanisms engage, cheapest first:

1. **Defragment** (`defragmenter_oom_policy`) — maybe it's not really out of memory, just a
   fragmented CUDA pool; `cudaMemPoolTrimTo()` and retry.
2. **Downgrade / spill** (`downgrade_executor`) — one per memory space. The GPU executor
   issues **one** `request_downgrade(predicate)` ("free until `make_reservation_or_null`
   succeeds"); the executor moves *idle* GPU batches to host/disk, **predicate-bounded** so
   it frees *just enough*. Candidates come first from inter-pipeline repository batches,
   then from queued pipeline tasks (which is *why* the scheduler keeps tasks
   downgrade-visible in its queue — the property the pull-signal matcher preserves,
   [`../file-maps/pipeline/task_scheduler.md`](../file-maps/pipeline/task_scheduler.md)).
3. **OOM-as-resumable-retry** — if an allocation *still* fails mid-execution, the operator
   throws `oom_reschedule_exception` carrying a resume operator index, and the GPU executor
   reschedules the task to continue from where it stopped (up to `MAX_OOM_RETRIES = 100` —
   *not* the doc's 10), [`../file-maps/pipeline/gpu_pipeline_executor.md`](../file-maps/pipeline/gpu_pipeline_executor.md).

Together: defrag → spill → retry, before anything fails. This is the "correctness across
the memory layer" half of the checkpoint — and it's full of async-lifetime and
locking subtleties (idle-only spill candidates, RAII task return-to-queue, teardown syncs)
that are exactly where bugs hide.

## Part 3 — When the GPU *can't* run it at all: fall back to CPU

**File map:** [`../file-maps/fallback.md`](../file-maps/fallback.md) · **docs:**
`dynamic-filters.md`, `optimizations.md`

Spilling handles "too big for GPU memory." **Fallback** handles "the GPU path can't run
this query (or this operator) at all." Two flavors:

- **Flavor 2 (reject):** unsupported features throw `NotImplementedException` from the plan
  generator / operators / expression translator — the consolidated "when does Sirius fall
  back?" list (DISTINCT aggregates, non-fixed-width casts, BIGINT `SUM`, non-`BOUND_REF`
  TOP-N keys, window/ASOF, …).
- **Flavor 1 (replay):** the extension layer catches plan/execution failures and re-runs
  the **original SQL on DuckDB CPU** (`run_internal_cpu_fallback_query` in
  [`../file-maps/sirius_extension.md`](../file-maps/sirius_extension.md)) — **except S3**,
  which has no CPU fallback and errors instead.

> **Two corrections worth carrying** (both flagged in the fallback map): (1) the plan's
> `src/fallback.cpp` is stale — that file is **legacy `gpu_processing`** code; the live
> fallback is the mechanism above. (2) `dynamic-filters.md` describes **scaffolding only**
> — no producers/consumers wired yet — so read it as design intent. `optimizations.md` is a
> *catalog* indexing files you've already mapped (partition, sort, hash join, task creator).

## Part 4 — Multi-GPU (skim)

**doc:** `docs/super-sirius/multi-gpu-architecture.md` (skim unless your work touches it)

You've already absorbed the load-bearing multi-GPU facts in earlier maps: the
**per-task-device colocation contract** (every input batch arrives on the task's
reservation device before `execute()`), device-preference routing in the scheduler, and
per-GPU stream pools (the scan executor's FIX-01). The multi-GPU doc fills in the topology
and distribution policy; read deeply only if you take on a multi-GPU task.

## Part 5 — Week 7: the non-trivial contribution (workflow)

No new file map — this is process, and you now have the surface area to pick something
real. Good non-trivial targets, each grounded in a layer you've mapped:

- **Extend an operator's supported cases** → move something from *flavor-2 reject* to
  GPU-native (e.g. a TOP-N ordering expression, an aggregate case). Touches the operator +
  expression layers (Week 3).
- **Add/extend an operator** → follow "Adding New Operators" in `CLAUDE.md`; you'll touch
  the plan generator, the operator base, the task-creation overrides (Weeks 2–5).
- **Improve a scheduling/memory path** → a downgrade candidate-selection tweak, a
  reservation-sizing refinement (Week 6) — the deep end, where the checkpoint's "reason
  about correctness across the concurrency and memory layers" really gets tested.

Process discipline:
- **`/module-context` before implementing** — loads the cuDF/RMM/cuCascade/DuckDB API docs
  so signatures are right (see the note at the top of
  [`../onboarding-path.md`](../onboarding-path.md); solo, open
  `.claude/skills/module-discover/docs/<lib>/modules/`).
- Follow `CLAUDE.md`'s operator recipe; write **both** `test/cpp/` and `test/sql/` tests
  (prime with [`../reference/explainers/testing-sirius.md`](../reference/explainers/testing-sirius.md)).
- `pre-commit run -a`, then `/code-review` before pushing.

## Where this leaves you

You can now reason about a query end-to-end *and* under stress: reserved before it runs,
spilled (defrag → downgrade) when the GPU fills, retried from a resume point on OOM, and
replayed on CPU when the GPU path can't take it — across one or many GPUs, without races.
That's the full picture the onboarding path set out to build, and the basis for making
non-trivial, correctness-aware contributions. The conceptual reading thread
([`../reference/explainers/README.md`](../reference/explainers/README.md)) and the code
spine ([`../file-maps/README.md`](../file-maps/README.md)) are your two maps back into any
of it.
