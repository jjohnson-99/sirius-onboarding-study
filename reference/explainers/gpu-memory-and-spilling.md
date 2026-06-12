# GPU Memory Management & Spilling

A GPU's bandwidth is its superpower; its **memory capacity is its weakness**. A dataset
(or even one query's intermediates) routinely exceeds the GPU's tens of GB, so a GPU SQL
engine lives or dies on how it manages and *spills* memory. This is one of the genuinely
hard, novel parts of Sirius — the cuCascade tiered-memory + reservation + downgrade
machinery — and the flip side of the bandwidth story in
[`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md).

> **Prime around:** Week 6 — `docs/super-sirius/memory-management.md`, `src/memory/`,
> `src/downgrade/`, `src/fallback.cpp`. It builds on
> [`cuda-streams-and-async.md`](cuda-streams-and-async.md) (stream-ordered allocation,
> async spill copies).

## Why GPU memory is the hard constraint

- **Small.** GPU memory (HBM) is on the order of tens of GB; host RAM is hundreds of GB to
  TB, and data can be larger still. You *cannot* assume everything fits on the GPU.
- **Slow to allocate.** Raw `cudaMalloc` is expensive and synchronizes the device — calling
  it per-operation would wreck throughput.
- **No good paging.** A CPU transparently pages to disk via virtual memory; a GPU has no
  equivalent that's fast enough for DB work (Unified Memory page-faults, but it's slow for
  scans). So the engine must manage capacity **explicitly**.

Two problems follow: *how to allocate cheaply* (pooling) and *what to do when you run out*
(tiering + spilling).

## Allocation: RMM pooling

[`cuda-streams-and-async.md`](cuda-streams-and-async.md) introduced stream-ordered
allocation. **RMM** (RAPIDS Memory Manager) grabs a big slab of GPU memory **once** and
sub-allocates from a **pool** — turning per-op `cudaMalloc` (slow, synchronizing) into
cheap pool hits. cuDF allocates through an RMM `memory_resource`, which is the allocator
threaded into every operator's `execute()`. Pooling is the baseline that makes GPU work
fast; tiering is what happens when the pool can't grow.

## The tiered hierarchy (cuCascade)

Sirius (via cuCascade) treats memory as **tiers**, and a data batch (a `cudf::table`
wrapped in a `data_batch`) can live in — and move between — any of them:

```
GPU  (HBM)         fastest, smallest    ◀─ tasks compute here
 ▼  downgrade (spill)  ▲ promote (lock back)
HOST (pinned RAM)  slower, bigger        ◀─ spill target; pinned for fast PCIe
 ▼                     ▲
DISK (NVMe)        slowest, huge         ◀─ last resort when even host fills
```

"Spilling" = pushing a batch *down* a tier (GPU→host→disk) to free scarce GPU memory;
"promotion" = pulling it back *up* into GPU when a task needs it. The **host tier is
pinned** (page-locked) so those transfers hit full PCIe bandwidth and overlap with compute
(the pinned-memory point from the streams explainer).

## Reservations: admission control so tasks don't OOM mid-flight

The central Sirius mechanism: **before** a GPU task is dispatched, the engine **estimates
its peak GPU memory** and acquires a **reservation** from the
`sirius_memory_reservation_manager` for that memory space. If the space can be reserved,
the task runs knowing it has room; if not, the task waits and/or the system spills first.
This is **admission control** — don't launch work you can't fit — and it prevents the
thrash-and-OOM spiral.

The estimate comes from the operator interface you saw in
[`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md):
`no_history_peak_memory_estimate()` (a first-run guess, default ~2× input bytes) and
`get_estimated_size_in_bytes()`; later runs refine it from execution history.

## Spilling: the downgrade executor

Reservations bound *new* work, but intermediate results still pile up in the data
repositories between pipelines ([`push-vs-pull-execution.md`](push-vs-pull-execution.md)).
A **`downgrade_executor`** (one per memory space) handles the overflow:

- a **monitor thread** polls GPU memory pressure (~every 10 ms);
- when a high-water threshold is crossed, it picks spill-candidate batches (idle
  intermediates in repositories) and schedules **downgrade tasks**;
- worker threads **async-copy** those batches GPU→host (D2H on a stream) and free the GPU
  buffers — making room *without* failing the query.

When a downgraded batch is needed again, it's **promoted/locked back** into GPU before the
consuming task runs (`prepare_for_processing` / `lock_or_prepare_batch` convert-then-lock a
batch into the requested space). So GPU↔host movement is continuous and demand-driven, not
a one-shot.

## Why in-use data can't be spilled: read-only locks

Spilling concurrently with execution is only safe because a `data_batch` has **states** and
**RAII locks**. When a task consumes a batch, it takes a **`read_only_data_batch`** lock
that pins the batch in GPU memory for the duration; `remove_read_only_lock()` releases it.
The downgrade executor only spills **unlocked, idle** batches — it can't pull data out from
under a running kernel. (Same `prepare_for_processing` machinery from the operator base.)

## When spilling isn't enough: OOM retry & CPU fallback

- **OOM retry.** Estimates can be too low. If a task throws `oom_reschedule_exception`, the
  GPU executor catches it and **retries** (up to ~10×, with backoff) while downgrade frees
  memory — graceful degradation instead of a hard crash.
- **CPU fallback.** If the data simply won't fit the GPU+host+disk budget (or hits another
  unsupported case), Sirius falls back to DuckDB CPU execution (`src/fallback.cpp`) — the
  ultimate safety valve.

## Multi-GPU & NUMA

Tiers are **per memory space**: each GPU has its own reservation manager and downgrade
executor, and the host tier is **NUMA-local** (per-NUMA pinned pools) so spills land on RAM
close to the right GPU. Cross-GPU movement uses P2P + the stream-lineage ordering from
[`cuda-streams-and-async.md`](cuda-streams-and-async.md).

## Quick glossary

| Term | One-liner |
|---|---|
| **RMM pool** | grab GPU memory once, sub-allocate cheaply (avoids per-op `cudaMalloc`) |
| **Tier** | GPU / host(pinned) / disk — where a batch can live |
| **Reservation** | pre-dispatch guarantee of GPU space (admission control) |
| **Peak estimate** | predicted max GPU bytes a task needs (sizes the reservation) |
| **Downgrade / spill** | move a batch GPU→host(→disk) to free GPU memory |
| **Promote** | move a spilled batch back up into GPU for a task |
| **Read-only lock** | RAII pin keeping an in-use batch resident (un-spillable) |
| **OOM reschedule** | catch out-of-memory, free via spill, retry the task |
| **Memory space** | a per-GPU (NUMA-aware) tiered allocation domain |

## See also

- [`cuda-streams-and-async.md`](cuda-streams-and-async.md) — stream-ordered allocation and
  the async copies spilling uses.
- [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) — GPU memory as the scarce
  resource (the flip side of bandwidth).
- [`file-maps/sirius_context.md`](file-maps/sirius_context.md) — owns the memory-reservation
  manager and the per-space downgrade executors;
  [`push-vs-pull-execution.md`](push-vs-pull-execution.md) — the repositories where
  intermediates (spill candidates) live.
