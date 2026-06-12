# cuDF (the GPU primitive library)

`cudf::` calls are everywhere in Sirius — every operator's `execute()` makes one — but
what *is* cuDF? Short version: it's the GPU library Sirius is built on.

> **Prime around:** Week 2 — you hit `cudf::` in the first operator you read. Use
> `/module-context` to load the cuDF module docs when a call is unfamiliar.

## What it is

**cuDF = CUDA DataFrame**, part of NVIDIA's **RAPIDS** suite. Two layers:

- **`cudf`** (Python) — a pandas-like GPU DataFrame API.
- **`libcudf`** (C++) — the engine underneath it. **Sirius links against libcudf**
  (`cudf::cudf`).

libcudf provides two things:

1. **GPU columnar data structures** — `cudf::column` (a contiguous typed device array +
   null mask), `cudf::table` (a set of columns), and non-owning `column_view` /
   `table_view`. These are the types in every Sirius `execute()` and in the
   [`duckdb-types-glossary.md`](reference/duckdb-types-glossary.md)'s neighborhood. Arrow-style
   columnar layout ([`columnar-vs-row-storage.md`](columnar-vs-row-storage.md)), allocated
   via RMM ([`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md)), run on CUDA streams
   ([`cuda-streams-and-async.md`](cuda-streams-and-async.md)).
2. **A library of GPU-accelerated relational primitives** — joins, group-by, sort, reduce,
   filter/compaction, gather, AST expression eval, string ops, Parquet I/O — each a tuned
   CUDA kernel.

## Why it's central to Sirius

Sirius doesn't re-implement GPU joins or sorts — it **orchestrates libcudf**. A Sirius
operator is largely a thin translator from a DuckDB plan node to one or a few libcudf
calls over `cudf::table`s:

| SQL / operator | libcudf primitive |
|---|---|
| JOIN | `cudf::hash_join`, `cudf::inner_join`, `cudf::mixed_join` |
| GROUP BY / aggregate | `cudf::groupby`, `cudf::reduce` |
| ORDER BY / TOP-N | `cudf::sort` / `order_by`, `cudf::top_k_order`, `cudf::merge` |
| WHERE | `cudf::apply_boolean_mask` (+ `cudf::ast` for the predicate) |
| LIMIT | `cudf::slice` |
| scan | `cudf::io::read_parquet` |

So "Sirius runs SQL on the GPU" largely means "Sirius maps each SQL operator onto a
libcudf primitive." It writes its own CUDA kernels only where libcudf doesn't fit —
e.g. the `src/cuda/scan/` Parquet decoders and custom top-N.

cuDF functions take a `(stream, memory_resource)` pair — the
[`cuda-streams-and-async.md`](cuda-streams-and-async.md) / RMM pattern — which is exactly
the `rmm::cuda_stream_view` + allocator threaded through Sirius operators.

## See also

- [`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md) —
  where `execute()` calls cuDF; and the operator maps for the concrete calls.
- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the columnar layout cuDF
  embodies; [`gpu-warps-and-execution.md`](gpu-warps-and-execution.md) — what runs inside a
  cuDF kernel.
