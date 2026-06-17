# NUMA (Non-Uniform Memory Access)

On a laptop, every core reaches every byte of RAM equally fast, so you never think about
*where* memory lives. On the big multi-socket servers Sirius targets, that stops being true —
and the difference between "memory attached to my socket" and "memory attached to the other
socket" is large enough that a bandwidth-bound engine has to manage it on purpose. That's what
NUMA is about.

> **Prime around:** Weeks 5–6 — when you read the host-memory tier (spilling, pinned pools)
> and the scan I/O path (host→GPU copies). It's a *GPU-systems* supporting concept, not part
> of the core "why GPU-native SQL" arc.

## The core idea: "non-uniform" = location matters

A small machine is **UMA** (uniform): one memory pool, every core equidistant. A multi-socket
server is **NUMA**: each CPU socket has its **own** memory controller and its **own** bank of
RAM directly attached. It's still one shared address space — any core can read any byte — but:

- accessing RAM on **your own socket** (your *NUMA node*) is fast — **local** access;
- accessing RAM on **another socket** crosses an inter-socket interconnect (Intel UPI / AMD
  Infinity Fabric) — higher latency, lower bandwidth — **remote** access.

```
   Socket 0  (NUMA node 0)              Socket 1  (NUMA node 1)
  ┌────────────────┐                   ┌────────────────┐
  │ cores 0–15     │── local ─▶ [RAM0] │ cores 16–31    │── local ─▶ [RAM1]
  └───────┬────────┘                   └───────┬────────┘
          └─────────── UPI / Infinity Fabric ──┘     ← remote: slower, less bandwidth
   core 3 → RAM0 : fast   (local)
   core 3 → RAM1 : slow   (remote — hops the interconnect)
```

So a 2-socket box typically presents **2 NUMA nodes**, each with a slice of the cores and a
slice of the RAM. "Non-uniform" just names the fact that your effective latency/bandwidth
depends on the core↔memory pairing.

| | UMA (laptop / 1 socket) | NUMA (multi-socket server) |
|---|---|---|
| Memory pools | one, shared | one per socket (node) |
| Access cost | uniform | local = fast, remote = slow |
| Address space | single | single (still!) — only *cost* differs |
| Who cares | nobody | bandwidth-bound systems (databases) |

## Why a database has to care

Analytics is **bandwidth-bound** (see [GPUs vs. CPUs](gpu-vs-cpu-for-databases.md)) — the
engine's speed is gated by how fast it streams bytes. Remote NUMA access can cost a large
fraction of bandwidth plus extra latency, so a thread on node 0 hammering memory that physically
lives on node 1 silently runs well under peak. The remedy is **NUMA-aware placement**: pin a
worker to a node *and* allocate the memory it touches on that same node, keeping the hot path
local. The OS exposes the controls (`numactl`, `libnuma`, `mbind`); high-performance engines
manage it explicitly rather than leaving placement to luck.

A useful mental rule: **co-locate the three things that move together — the thread, its memory,
and (for a GPU box) the device.** Each GPU also hangs off a particular socket via PCIe, so "feed
GPU 1 from RAM on GPU 1's socket" is the same locality principle one level up. A cross-node H2D
copy pays the interconnect tax *before* it even reaches PCIe.

## Where NUMA shows up in Sirius

Sirius is built for multi-socket + multi-GPU servers, so the **host-memory** tier and the
host→GPU transfer path are both NUMA-aware. This is why `numa_id` keeps appearing:

- **Per-NUMA pinned host allocator — `numa_small_pinned_mr`** (`src/include/memory/numa_small_pinned_mr.hpp`).
  It owns **one `small_pinned_host_memory_resource` per NUMA node** and routes each
  allocate/deallocate by reading `cudaGetDevice()` and looking the device up in a
  device-id→NUMA-node map (with a ptr→node table so frees return to the originating node's pool).
  Its header spells out the bug it fixes: without it, *every* GPU's cuDF metadata allocation
  funneled through the NUMA-0 host space, defeating the per-NUMA layout. Owned by `SiriusContext`
  as `small_pinned_allocator_` ([`sirius_context.md`](../../file-maps/sirius_context.md)).
- **Host memory spaces carry a `numa_id`** — the per-node affinity in the host `memory_space`
  config (the tiered host pools; see [`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md)
  / the memory-management doc). Pinned staging pools are placed per node.
- **NUMA-aware H2D copy hints in the scan path** — `cache_ranges` carries a `numa_id` hint
  (`-1` = unset) for batch copies (`src/include/op/scan/cached_ranges.hpp`), and the cached
  datasource sets it on the copy: `attr.srcLocHint = {cudaMemLocationTypeHost, numa_id}`
  (`src/op/scan/prefetched_data_source.cpp:148`) so a batched H2D lands from node-local memory.
  Ties into the scan executor's memory-proportional GPU targeting
  ([`duckdb_scan_executor.md`](../../file-maps/op/scan/duckdb_scan_executor.md)).
- **Per-NUMA I/O reactors** — the io_uring stack (`src/io/uring/`) runs reactors per node so file
  reads land in node-local pinned buffers before going to the GPU (the `sirius::io` substrate
  behind [Parquet](parquet-format.md) scanning).

The throughline: pinned host memory is the staging ground for every GPU transfer (see the
pinned-memory note in [CUDA streams & async](cuda-streams-and-async.md)), so Sirius wants that
staging buffer on the **same NUMA node as the GPU that will consume it** — keep thread, host
memory, and device on one node and you avoid paying the interconnect tax on every batch.

## Relationship to GPU memory (don't conflate them)

NUMA is a **host (CPU) RAM** concept — it's about which socket owns a stretch of system memory.
It's *not* about GPU device memory (that's RMM/cuCascade tiers; see
[GPU memory & spilling](gpu-memory-and-spilling.md)). They meet only at the **transfer**: the
host side of an H2D copy / a spill-to-host should be node-local to the GPU. Think of NUMA as the
geography of the *host* tier that the GPU reaches across PCIe.

## Takeaway

NUMA = multi-socket machines where RAM access is fast if local to your socket and slow if
remote, because each socket owns its own memory. A bandwidth-bound GPU database fights this by
keeping each worker's thread, its host memory, and its GPU on the same node. In Sirius that's
the `numa_small_pinned_mr` (per-node pinned allocator), per-node host memory spaces, the
`numa_id` copy hints in the scan path, and per-NUMA io_uring reactors — all so the host side of
every GPU transfer stays local.
