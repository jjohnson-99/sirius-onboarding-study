# GPUs vs. CPUs for Databases

[`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) claims you "cannot run an
efficient GPU kernel over row-major tuples." That's not hyperbole — it follows directly
from how a GPU is built, and understanding why is the key to understanding *why Sirius
exists at all*. A GPU isn't just "a CPU with more cores"; it's a throughput machine whose
speed is gated by **memory-access patterns**, and columnar is what feeds it.

> **Prime around:** Week 1–2 — foundational background for the whole project; most
> concrete when you first read an operator's `execute()` calling a `cudf::` kernel
> ([`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md))
> and the `src/cuda/scan/` decoders.

## Two different design goals: latency vs. throughput

| | CPU | GPU |
|---|---|---|
| Cores | few, complex (4–64) | thousands, simple |
| Optimized for | **latency** — finish one thread fast | **throughput** — finish many threads/sec |
| Per-core tricks | out-of-order, branch prediction, deep caches, prefetchers | almost none — simple in-order lanes |
| Parallelism | a handful of threads, each with wide SIMD (8–16 lanes) | tens of thousands of threads in flight |
| Memory | ~50–100 GB/s DRAM, big caches | ~1–3 **TB/s** HBM, tiny per-thread cache |
| Latency handling | hide it with caches + OoO | hide it by **switching to another warp** |

The CPU spends transistors making *one* thread fast and forgiving (caches and prefetchers
paper over messy access patterns; branch predictors paper over branchy code). The GPU
spends transistors on *many* simple lanes and enormous memory bandwidth, and pushes the
burden of feeding them efficiently onto the **data layout**.

## SIMT: threads run in lockstep warps of 32

GPU threads execute in **warps** of 32, SIMT-style (Single Instruction, Multiple
Threads): all 32 run the *same* instruction at the same time, each on its own data
element. For a database scan you map **thread `i` → element `i` of a column** — so a warp
processes 32 consecutive column values in one step. Two things break this:

- **Branch divergence.** If threads in a warp take different paths (`if` on per-row data),
  the warp executes *both* paths serially with masking — you lose parallelism. Uniform,
  per-column processing minimizes this.
- **Scattered memory access** — the big one, next.

## Memory coalescing: the crux

GPU global memory is read in **fixed-size segments** (e.g. 32- or 128-byte transactions),
not individual bytes. The hardware is fast *only* when a warp's 32 threads touch
**contiguous** addresses, so they fold into **one** transaction. That's *coalescing*.

**Columnar (contiguous) → coalesced → full bandwidth:**

```
column qty: [ q0 q1 q2 q3 ... q31 q32 ... ]   (one packed int32 array)
warp reads   t0 t1 t2 t3 ... t31              → 32 ints = 128 contiguous bytes
                                              → ONE memory transaction, 100% useful
```

**Row-major (strided) → uncoalesced → bandwidth thrown away:**

```
rows: [id0 name0 qty0][id1 name1 qty1][id2 name2 qty2]...   (each row e.g. 64 bytes)
warp reads qty0, qty1, qty2, ... qty31  → addresses 64 bytes apart
                                        → 32 SEPARATE transactions; each fetches a
                                          128-byte segment to use 4 bytes → ~3% useful
```

To read one column from row-major data, each thread must **gather** its value from a
different row, strided by the row width. The 32 accesses scatter across memory, defeat
coalescing, and you fetch ~32× the bytes you use — collapsing the GPU's single biggest
advantage (that TB/s of bandwidth) to a fraction. That is the concrete reason a row-major
GPU kernel is inefficient: **not enough compute is the problem — wasted memory bandwidth
is.**

(A CPU survives the same strided access far better: its prefetcher and large caches absorb
it, and out-of-order execution hides the latency. So columnar is *better* on a CPU but
*mandatory* on a GPU — the GPU has none of those safety nets.)

## Why GPUs win for analytics anyway

Most database scan/filter/aggregate work is **memory-bandwidth-bound**, not
compute-bound. The GPU's HBM bandwidth (often 10–30× a CPU's DRAM) is exactly the resource
those operations need — *if* access is coalesced. Columnar layout + one-thread-per-element
+ coalesced reads is how you cash in that bandwidth. That's the whole bet of a GPU SQL
engine: turn bandwidth-bound relational ops into wide, coalesced, divergence-free passes
over contiguous columns.

Two more GPU-specific costs the design must respect:

- **The PCIe boundary.** Data must cross host→device (PCIe, ~tens of GB/s — slow relative
  to HBM). So you want to move data to the GPU **once** and do many ops there, never
  round-trip per operator. (This is why Sirius reads parquet column-chunk bytes onto the
  GPU and **decodes them there** — `src/cuda/scan/` — and keeps batches GPU-resident,
  spilling to host only under memory pressure.)
- **Kernel-launch overhead.** Each kernel launch costs microseconds, so each must do a lot
  of work. This is why GPU "batches" are huge (whole `cudf::table`s of thousands–millions
  of rows) rather than the CPU's ~2048-row vectors — see
  [`vectorized-execution.md`](vectorized-execution.md). SIMT is vectorization taken to the
  hardware extreme.

## How this shows up in Sirius

- A batch on the GPU is a `cudf::table` = a set of `cudf::column`s, each a **contiguous
  device array** — built for coalesced access. Converting DuckDB/parquet data into that
  layout is much of what the scan path does.
- Every operator's `execute()` hands `table_view`s to a cuDF primitive that launches a
  kernel assigning ~one thread per element over those contiguous columns
  ([`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md)).
- The parquet **GPU decoders** ([`parquet-format.md`](parquet-format.md)) exist so the
  data lands in coalesce-friendly columnar form on the device without a CPU round-trip.
- Memory tiers / downgrade exist because GPU memory is small and precious relative to the
  data — the flip side of all that bandwidth.

## See also

- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the layout this hardware
  demands.
- [`vectorized-execution.md`](vectorized-execution.md) — SIMT as the extreme of
  vectorized execution.
- [`parquet-format.md`](parquet-format.md) — why decoding happens on the GPU.
