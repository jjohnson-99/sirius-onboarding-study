# CUDA Streams & Async Execution

Every Sirius operator's `execute(input, rmm::cuda_stream_view stream)` takes a **stream**,
and the whole pipeline runs work asynchronously on the GPU. This explainer is what that
parameter means: how the CPU and GPU run concurrently, how independent GPU work overlaps,
and why pinned memory and `synchronize()` calls show up where they do. It's the async
companion to [`gpu-warps-and-execution.md`](gpu-warps-and-execution.md).

> **Prime around:** Week 2 — you meet `cuda_stream_view` in every `execute()`
> ([`file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md))
> — then Weeks 5–6, where async host↔device transfers and memory really lean on it.

## The async model: the GPU is a separate worker you queue work for

The CPU (host) and GPU (device) run independently. When the host launches a kernel or an
async copy, it **enqueues** the operation and **returns immediately** — it does *not* wait
for the GPU to finish. The host can keep doing other things (launch more work, prep the
next batch) while the GPU churns. GPU work is fire-and-forget until you explicitly wait.

## A stream is an ordered queue of GPU work

A **CUDA stream** is a FIFO queue of GPU operations (kernel launches, memory copies):

- Operations **within one stream** run **in order** — each waits for the previous.
- Operations in **different streams** have **no ordering** between them — they can run
  **concurrently / overlap** if the hardware has capacity and there's no data dependency.

So a stream is how you express "these ops are sequential," and *using multiple streams* is
how you expose independent work for the GPU to overlap. (The legacy **default/NULL stream**
is special — it synchronizes with all others — so high-performance code uses explicit
non-default streams, or per-thread default streams.)

## Synchronization: knowing when the GPU is done

Because launches are async, you need explicit waits:

- `cudaStreamSynchronize(stream)` — block the host until *that stream's* queued work
  finishes.
- `cudaDeviceSynchronize()` — block until *everything* finishes (the heavy hammer; avoid
  on hot paths).
- **Events** (`cudaEvent`) — record a marker in a stream; one stream can **wait on another
  stream's event** (`cudaStreamWaitEvent`) to express a cross-stream dependency **without
  blocking the host**, and events are also how you time GPU work. Events are the
  fine-grained tool; full sync is the blunt one.

## What overlap buys you: hide the PCIe cost

A GPU has separate **copy engines** and **compute engines**, so with multiple streams it
can do three things at once: **H2D copy**, **compute**, **D2H copy**. The classic win is
**copy/compute overlap** — process batch *N* while copying batch *N+1* in:

```
stream A:  H2D(b0)  ─ compute(b0) ─ D2H(b0)
stream B:           H2D(b1) ──────── compute(b1) ─ D2H(b1)
stream C:                    H2D(b2) ──────────────── compute(b2) …
            └ copy of b1/b2 overlaps compute of b0/b1 → PCIe latency hidden
```

This directly attacks the **PCIe boundary** cost from
[`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md): instead of "copy, then
compute, then copy" serially, transfers hide behind computation. Independent **kernels** on
different streams can also run concurrently when SMs have spare capacity.

## Pinned host memory (the prerequisite for async copies)

`cudaMemcpyAsync` is only *truly* asynchronous — and only reaches full PCIe bandwidth —
when the **host** buffer is **pinned** (page-locked). Pageable memory can't be DMA'd
directly, so the driver stages it through a pinned bounce buffer, serializing the copy. So
overlapping transfer with compute **requires pinned host buffers**. This is why Sirius
keeps pinned host memory (`small_pinned_host_memory_resource`, the `use_pin_memory`
config, and the NUMA-pinned bounce slots in `sirius_ioctx`) — it's the substrate that makes
async streaming copies work.

## Stream-ordered memory allocation (RMM)

Allocation itself is stream-ordered in modern CUDA (`cudaMallocAsync`-style pools): an
allocate/free is enqueued on a stream and ordered with the work around it, so memory can be
**reused safely within stream order without over-synchronizing**. **RMM** (RAPIDS Memory
Manager) provides exactly this — stream-ordered memory resources backed by a pool — and
**cuDF** functions take a `(stream, memory_resource)` pair. That's precisely the
`rmm::cuda_stream_view` + allocator you see threaded through Sirius operators: the op
enqueues its cuDF kernels *and* its allocations on the task's stream.

## How Sirius uses streams

- **One stream per task/worker.** GPU worker threads run pipeline tasks on dedicated
  streams, so independent tasks overlap; each operator's `execute()` enqueues its cuDF work
  on that task's `cuda_stream_view`. cuDF ops are stream-ordered, so a pipeline's operators
  chain correctly on one stream without per-op host syncs.
- **Scan overlap.** The scan path issues **async H2D copies** of parquet column-chunk bytes
  (through pinned bounce buffers) and overlaps them with decode/compute — copy/compute
  pipelining to hide PCIe (`docs/super-sirius/scan.md`; `opt_table_scan_num_streams`
  controls how many streams the optimized scan uses).
- **Sync at boundaries.** You'll see `stream.synchronize()` right before device buffers are
  freed or handed across a boundary (e.g. the pin-table D2H path), so async copies finish
  before their source is released.
- **Cross-stream / cross-device dependencies.** cuCascade tracks **stream lineage** — a
  `data_batch` records which stream/device *wrote* it, so a downstream reader on a different
  stream or GPU can `cudaStreamWaitEvent` on that producer instead of a global sync (the
  "STREAM-LINEAGE" comments in the aggregate operator). This is events-based cross-stream
  ordering at the data-flow level, and it's what makes multi-GPU correctness possible.

Honest framing: you rarely call `cudaMemcpyAsync` or manage streams by hand in Sirius —
cuDF, RMM, and cuCascade do — but the stream model explains *why* `execute()` takes a
stream, why pinned memory exists, why there are targeted `synchronize()` calls, and how the
engine hides transfer latency.

## Quick glossary

| Term | One-liner |
|---|---|
| **Stream** | ordered FIFO queue of GPU ops; different streams can overlap |
| **Async launch** | host enqueues work and returns immediately; GPU runs it later |
| **`cudaMemcpyAsync`** | non-blocking copy; needs **pinned** host memory to truly overlap |
| **Pinned (page-locked) memory** | host memory the GPU can DMA directly; required for async copies |
| **`cudaStreamSynchronize`** | block host until one stream's work is done |
| **Event** | marker in a stream; enables timing + cross-stream waits without host sync |
| **Stream-ordered allocation** | alloc/free enqueued on a stream (RMM pools); safe reuse without over-sync |
| **Copy/compute overlap** | transfer batch N+1 while computing batch N → hides PCIe |

## See also

- [`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md) — stream-ordered allocation in
  the bigger picture: pooling, tiers, and spilling under memory pressure.
- [`gpu-warps-and-execution.md`](gpu-warps-and-execution.md) — what runs *inside* a kernel
  (this is how kernels and copies are *scheduled*).
- [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) — the PCIe boundary that
  copy/compute overlap hides.
- [`parquet-format.md`](parquet-format.md) — the scan transfers/decodes that stream
  overlap accelerates.
