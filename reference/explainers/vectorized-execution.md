# Vectorized Execution

If columnar storage ([`columnar-vs-row-storage.md`](columnar-vs-row-storage.md)) is
about how data sits *still*, vectorized execution is about how it *moves* through the
engine — and it's the trick that lets an interpreted SQL engine run at close to
hand-written-loop speed. It's the processing model behind DuckDB's `DataChunk`
([`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md)) and the reason a Sirius
"operator" works on batches the way [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md)
describes.

> **Prime around:** Week 1 — read with the CMU "processing models" lecture and
> `week1-concepts`.

## The problem it solves

The textbook query engine is the **Volcano / iterator model**: every operator exposes
`next()`, which returns **one tuple**. A query is a tree of operators, so producing one
output row means a chain of `next()` calls rippling down the tree — one **virtual
function call per operator, per row**.

For `SELECT sum(l_quantity) FROM lineitem WHERE l_quantity > 10` over 6M rows, that's
tens of millions of virtual calls, plus per-value type dispatch, plus branch
mispredictions — and none of it vectorizes on the CPU's SIMD units. The *actual*
arithmetic is a rounding error next to the per-tuple bookkeeping. That overhead is the
problem.

## Three processing models

| Model | Unit of work | Per-call overhead | Intermediate memory |
|---|---|---|---|
| **Tuple-at-a-time** (Volcano/iterator) | 1 row | huge (per row) | tiny |
| **Operator-at-a-time** (full materialization) | the *whole* column | negligible | huge (full intermediates) |
| **Vectorized** (batch) | a **batch** of ~1–2048 rows | amortized across the batch | small (cache-resident) |

Vectorized execution is the **sweet spot** between the two extremes: process a *vector*
(a batch of values) per call. One virtual call now covers ~2048 rows instead of one, and
the inner loop over the batch is a tight, branch-predictable, **SIMD-friendly** array
loop the compiler can auto-vectorize:

```cpp
// Volcano: one value, behind a virtual call, per row
Value next() { return a.next() + b.next(); }      // millions of virtual calls

// Vectorized: one virtual call, a tight loop over the whole vector
void execute(Vector& a, Vector& b, Vector& out, idx_t n) {
    for (idx_t i = 0; i < n; i++) out[i] = a[i] + b[i];   // SIMD-able, ~2048 at a time
}
```

Same answer; the second form moves the per-call overhead from *per row* to *per ~2048
rows* and lets the hardware do what it's good at.

## Why ~2048 (and not 1, and not everything)

The batch size is a deliberate cache trade-off:

- **Too small (1 = Volcano):** call overhead dominates.
- **Too big (whole column = full materialization):** the intermediate vectors blow past
  L1/L2 cache and you pay memory bandwidth shuttling them in and out.
- **Just right (~1–2048):** big enough to amortize the call and fill SIMD lanes, small
  enough that the few columns you're touching stay resident in cache.

DuckDB's default is **2048 rows** (`STANDARD_VECTOR_SIZE`).

## Why it *requires* columnar

A "vector" is a **slice of one column** — a contiguous run of same-type values. That's
what makes the inner loop a typed array loop (SIMD on CPU; one thread per element on
GPU). Try to vectorize over row-major tuples and there's no contiguous typed array to
loop over. So vectorized execution sits *on top of* columnar storage — they're a
package, which is why DuckDB/cuDF/Sirius are all both. (See
[`columnar-vs-row-storage.md`](columnar-vs-row-storage.md).)

One-line mental model: **vectorized execution is the Volcano model with the unit of work
changed from "a tuple" to "a column-batch."** The operator tree and the `next()`/push
flow are the same shape; only the granularity changed.

## DuckDB specifics

DuckDB is vectorized and **push-based**: operators pass `DataChunk`s downstream. A
`DataChunk` is a bundle of column `Vector`s (~2048 rows) — literally "the vector of
vectorized execution" from the glossary. Each operator's job is "consume a `DataChunk`,
produce a `DataChunk`," looping over the column `Vector`s inside.

## Sirius / GPU: vectorization taken to the extreme

A GPU is a vectorized machine in hardware — thousands of cores running the same
instruction over different array elements (SIMT). So Sirius is vectorized *to the core*,
just with a much **wider** vector:

- The "batch" isn't ~2048 rows tuned to a CPU cache — it's an entire `cudf::table`
  (thousands to millions of rows), sized to **saturate the GPU and amortize kernel-launch
  overhead** (controlled by config like `scan_task_batch_size`). On a GPU you want the
  batch *big*, the opposite pressure from the CPU's cache limit.
- An operator's `execute()` (see
  [`file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md))
  receives `cudf::table_view`s and calls a cuDF primitive that launches **one kernel over
  the whole column** — e.g. `cudf::reduce(l_quantity, SUM)` is a single massively-parallel
  pass, and the limit operator's `cudf::slice`
  ([`file-maps/op/sirius_physical_limit.md`](../../file-maps/op/sirius_physical_limit.md))
  is the same idea. That *is* a vectorized operator, with the vector = a GPU column.

So when Week 1 says Sirius "takes the vectorized idea to its limit," this is the precise
sense: the unit of work is a whole GPU column, and the inner loop is a CUDA kernel.

## See also

- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the storage layout
  vectorized execution requires.
- [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md) — processing models
  (Volcano vs. vectorized) and the operator model, in the Week 1 context.
- [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) — `DataChunk` / `Vector`.
- *Optional paper:* "MonetDB/X100: Hyper-Pipelining Query Execution" — the origin of
  vectorized execution (in the onboarding-path resource list).
