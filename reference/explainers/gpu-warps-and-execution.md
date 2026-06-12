# GPU Execution Model: Warps, Occupancy & Divergence

[`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) argues *why* a GPU suits
analytics; this is the deeper "how it actually executes" — the warp, the hardware it runs
on, and the handful of performance concepts and terms you'll keep meeting in cuDF, nsys
profiles, and the `src/cuda/` kernels.

> **Prime around:** Week 1–2 (with `gpu-vs-cpu-for-databases`) for the model; again at
> Week 5 / any time you read a `.cu` kernel or profile a query.

## The execution hierarchy (terminology)

A kernel launch fans out into a hierarchy:

```
grid  ─ the whole launch (all blocks of one kernel)
 └ block (CTA) ─ ≤1024 threads on ONE SM; can share memory + synchronize
    └ warp ─ exactly 32 threads, executed in lockstep (the scheduling unit)
       └ thread ─ runs the kernel on one element; has its own registers + index
```

- **Thread** — the kernel body executed for one data element; owns registers and a lane
  index. In a DB kernel, typically "thread *i* handles column element *i*."
- **Warp** — **32 threads** the hardware schedules and issues together (NVIDIA; AMD calls
  it a *wavefront*, 32 or 64). This is *the* unit that matters for performance.
- **Block / CTA (cooperative thread array)** — up to 1024 threads that live on one **SM**,
  can share **shared memory** and barrier-synchronize (`__syncthreads()`). Made of several
  warps.
- **Grid** — all the blocks of the launch, configured as `<<<gridDim, blockDim>>>`.
- **SM (Streaming Multiprocessor)** — the physical engine that runs blocks. Contains warp
  **schedulers** (often 4), a large **register file**, **shared memory / L1**, and the ALU
  lanes ("**CUDA cores**"). A GPU has tens to >100 SMs.

## SIMT: one instruction, 32 threads

A warp executes **SIMT** — Single Instruction, Multiple Threads: all 32 threads run the
**same instruction** each cycle, each on its own registers/data. It's SIMD under the hood,
but with a per-thread programming model (each thread has an index and *can* branch — at a
cost, see divergence). The warp scheduler issues one instruction for all 32 lanes at once.

## The core performance idea: hide latency by switching warps

This is the thing that makes a GPU a GPU. An SM keeps **many warps resident at once**
(e.g. up to ~64 warps / 2048 threads). Each cycle the scheduler picks a warp that's
**ready** and issues its next instruction. When a warp does a global-memory load — *hundreds*
of cycles of latency — the SM doesn't stall; it **switches to another ready warp**:

```
cycle →   W0 issue load ─(stalled, waiting on memory)──────────► ready again
          W1 issue        W2 issue        W3 issue   W0 resumes …
          (the scheduler always has *some* ready warp → SM stays busy)
```

So a GPU hides memory latency with **massive multithreading + zero-overhead warp
switching**, not with caches and out-of-order execution like a CPU. The consequence:
**you need lots of parallel work (many warps) to keep the SM fed.** Too few warps → latency
is exposed → the SM idles. This is *why* DB kernels want huge batches (millions of
elements = tens of thousands of warps): enough to saturate every SM.

## Occupancy

**Occupancy** = resident warps ÷ the SM's max. More resident warps = more latency to hide
behind. It's capped by per-SM **resources**:

- **Registers per thread** — the register file is finite; a register-hungry kernel fits
  fewer warps.
- **Shared memory per block** — finite per SM; greedy blocks reduce how many fit.
- **Threads/block** limits.

Higher occupancy usually helps, but it's not the only lever (memory access pattern and
instruction-level parallelism matter too) — "100% occupancy" isn't the goal, "enough warps
to hide latency" is.

## Warp divergence

Because a warp shares **one** instruction stream, if its threads take different branches
(`if (data[i] > x) …`), the warp executes **both** paths serially, masking off the lanes
not on the current path — then reconverges. Divergence can halve throughput (worse if
nested). Minimize it by making threads in a warp follow the **same** control flow.

For databases this is concrete: uniform fixed-width **column** processing is
divergence-free (every lane does the same op); **per-row branchy** logic, variable-length
strings, and ragged data are the divergence sources. (Volta+ "independent thread
scheduling" relaxes reconvergence rules but divergence still costs.)

## Memory: coalescing, hierarchy, bank conflicts

- **Coalescing** (the big one, detailed in
  [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md)): global memory is served in
  ~32–128-byte transactions. When a warp's 32 threads hit **consecutive, aligned**
  addresses, they fold into the minimum transactions (full bandwidth); strided/scattered
  access explodes into many transactions (wasted bandwidth). Contiguous columns coalesce;
  row-major gathers don't.
- **Memory hierarchy** (fast → slow): **registers** (per-thread) → **shared memory / L1**
  (per-block on-chip scratchpad, programmer-managed, for tiling and cross-lane sharing) →
  **L2** (shared across SMs) → **global / HBM** (large, ~TB/s bandwidth but hundreds of
  cycles latency).
- **Shared-memory bank conflicts**: shared memory is split into 32 banks; if lanes in a
  warp touch different addresses in the *same* bank, the accesses serialize. Padding /
  access patterns avoid it.

## Warp-level primitives

High-performance kernels exploit the warp directly: **shuffle** (`__shfl_sync` — lanes
exchange registers with no shared memory), **vote** (`__ballot_sync`, `__any/all_sync`),
and warp **reductions/scans**. A GPU reduction (e.g. a `SUM` aggregate) is typically
*warp-reduce via shuffles → block-reduce via shared memory → grid-combine via atomics or a
second pass*. You don't write these in Sirius day-to-day — cuDF does — but the
`src/cuda/scan/` decoders use warp cooperation (bit-unpacking, RLE), and profiles report
metrics in these terms.

## What it means for Sirius

- A cuDF kernel maps ~one thread per column element; a warp processes 32 consecutive
  elements with **coalesced** loads and **no divergence** (uniform op) — the ideal case the
  columnar layout is designed to produce.
- Sirius's large GPU batches exist partly to supply **enough warps to saturate the SMs**
  and amortize launch overhead (occupancy + the kernel-launch point from
  [`vectorized-execution.md`](vectorized-execution.md)).
- Divergence-prone work (strings, variable-length, branchy predicates) is where GPU DB
  performance gets hard — worth recognizing when you read those kernels or profiles.

## Quick glossary

| Term | One-liner |
|---|---|
| **Thread** | one kernel instance over one element; owns registers + index |
| **Warp** | 32 threads issued in lockstep; the scheduling unit |
| **Block / CTA** | warps on one SM that share memory + can synchronize |
| **Grid** | all blocks of a kernel launch |
| **SM** | the physical multiprocessor: schedulers, registers, shared mem, ALU lanes |
| **CUDA core** | one ALU lane |
| **SIMT** | one instruction, 32 threads, each on its own data |
| **Occupancy** | resident warps ÷ max; how much latency you can hide |
| **Divergence** | a warp's threads taking different branches → serialized paths |
| **Coalescing** | a warp's contiguous accesses folded into minimal memory transactions |
| **Shared memory** | per-block on-chip scratchpad (banked) |
| **Shuffle / vote** | warp-level register exchange / predicate polling primitives |

## See also

- [`cuda-streams-and-async.md`](cuda-streams-and-async.md) — how kernels and copies are
  *scheduled* (async, overlapped) — the complement to this "what runs inside a kernel."
- [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) — *why* GPUs suit
  bandwidth-bound DB work (this is the *how*).
- [`vectorized-execution.md`](vectorized-execution.md) — SIMT as vectorized execution in
  hardware; why batches are huge.
- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the layout that makes warp
  accesses coalesce.
