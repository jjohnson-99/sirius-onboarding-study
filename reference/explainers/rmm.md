# RMM (RAPIDS Memory Manager)

The `rmm::cuda_stream_view` in every operator's `execute()` is RMM's, and so is the pool
that makes GPU allocation fast. **RMM is the allocation layer** beneath both cuDF
([`cudf.md`](cudf.md)) and cuCascade ([`cucascade.md`](cucascade.md)) — the third NVIDIA
library Sirius is built on. This is the library behind the pooling concept in
[`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md).

> **Prime around:** Week 2 — you meet `rmm::cuda_stream_view` in the first operator — then
> Week 6 (memory pools / resources). `src/memory/`.

## What it is

RMM (RAPIDS Memory Manager) is an NVIDIA C++ library for **GPU memory allocation**. It
exists because raw `cudaMalloc`/`cudaFree` are slow and synchronize the device — calling
them per operation would wreck throughput. RMM provides a **pluggable allocator interface**
plus fast **pooling** and **stream-ordered** allocation.

## Core abstractions

| Type | What it is |
|---|---|
| **`device_memory_resource`** (MR) | the pluggable allocator interface: `allocate(bytes, stream)` / `deallocate(...)`. Everything else is an implementation or wrapper of this. |
| **`pool_memory_resource`** | grabs one big slab up front and **sub-allocates** from it — turns per-op `cudaMalloc` into cheap pool hits. **The one that matters for performance.** |
| **`cuda_memory_resource`** / **`cuda_async_memory_resource`** | thin wrappers over `cudaMalloc` (slow) / `cudaMallocAsync` (stream-ordered driver pool). |
| **adaptor MRs** | wrap another MR to add behavior: `statistics_`, `logging_`, `limiting_`, `tracking_resource_adaptor` (Sirius logs host-pool stats this way). |
| **`device_buffer` / `device_uvector` / `device_scalar`** | RAII owning device-memory containers built on an MR. |
| **`rmm::cuda_stream_view` / `cuda_stream`** | RMM's CUDA stream wrappers — *the* `cuda_stream_view` threaded through every Sirius `execute()`. |
| **`device_async_resource_ref`** | a type-erased handle to an MR, passed to cuDF (so cuDF allocates from *your* pool). |
| **`cuda_set_device_raii` / `cuda_device_id`** | RAII GPU-device switching — used on Sirius's multi-GPU paths. |

## Stream-ordered allocation

RMM allocations are **stream-ordered**: `allocate(bytes, stream)` and the matching free are
ordered relative to the work on that stream, so memory can be **reused safely within stream
order without over-synchronizing** (the model from
[`cuda-streams-and-async.md`](cuda-streams-and-async.md)). That's why cuDF takes a
`(stream, memory_resource)` pair — and why Sirius operators thread a stream + allocator
through every call.

## How Sirius (and cuDF) use it

- **cuDF allocates through an RMM MR.** You install a `pool_memory_resource` as the current
  device resource (`set_current_device_resource`), and cuDF's outputs come from that pool —
  fast, no per-op driver calls.
- **cuCascade builds on RMM.** A `memory_space` hands out the RMM allocator for its tier; the
  GPU tier is an RMM device pool, and Sirius also wires **pinned host** resources (e.g. the
  NUMA small-pinned MR) for the host tier and cuDF's pinned-memory path.
- **The stream everywhere.** `rmm::cuda_stream_view` is the stream type in `execute()`
  (see [`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md)),
  and `cuda_set_device_raii` switches GPUs for multi-GPU work.

## The layering (RMM, cuCascade, cuDF)

```
cuDF        ── compute over cudf::tables ── allocates via ▼
cuCascade   ── tiered/spillable data (GPU/host/disk) ── built on ▼
RMM         ── raw GPU allocation: pools, stream-ordered, MRs, device buffers
                (+ rmm::cuda_stream_view, the stream type used everywhere)
```

RMM is the floor: the fast allocator and the stream/device primitives. cuCascade adds
*where data lives*; cuDF adds *what to compute*. Together they're the three NVIDIA libraries
under Sirius.

## See also

- [`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md) — pooling/reservations/spilling
  concepts (RMM is the allocator underneath).
- [`cuda-streams-and-async.md`](cuda-streams-and-async.md) — stream-ordered allocation;
  [`cucascade.md`](cucascade.md) / [`cudf.md`](cudf.md) — the layers built on RMM.
