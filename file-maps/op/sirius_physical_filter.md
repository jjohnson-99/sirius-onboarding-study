# `op/sirius_physical_filter.cpp` → Operator Map

The standalone **FILTER** operator (`SiriusPhysicalOperatorType::FILTER`): drop rows that fail a
single boolean predicate. It is the thinnest possible wrapper around
[`gpu_expression_executor::select()`](../expression_executor/gpu_expression_executor.md) — construct
the executor from one AST predicate, run `select()` per input batch, wrap the surviving rows back
into batches. Paired with [`sirius_physical_projection`](sirius_physical_projection.md) (the
`execute()` sibling).

> Paths relative to the Sirius repo root; study-note links relative to this file. Lines as of
> 2026-07-01 — re-grep if the file moved. Tiny file (~65 lines); flat operator, no control-flow
> chain, so no call-tree (methods table below).

## Where this sits

A **regular streaming intermediate operator**: one input batch → one output batch, stateless, no
buffering. The plan generator builds it from DuckDB's `LOGICAL_FILTER` (see
[`../planner/plan-builders.md`](../planner/plan-builders.md)); at runtime a `gpu_pipeline_task`
calls its `execute()` like any other operator in the pipeline's operator loop.

```
pipelineable_operator_data (input batches)
   └─ execute(): for each batch
        gpu_expression_executor(*expression).select(table_view)   :56   fused mask + gather
        make_data_batch(filtered_table, mem_space, stream)        :59
   └─▶ pipelineable_operator_data (filtered batches)
```

> **Scan-path caveat.** This is the *textbook* FILTER. On the scan path the filter is **folded into
> the ingestible** (`duckdb_native_gpu_ingestible::post_filter_and_project`, or parquet reader
> pushdown) and this standalone operator is **not** used — the converter pushes `table_filters`
> into the `TABLE_SCAN`. This operator runs for filters that aren't pushed into a scan (e.g. a
> predicate over a join/aggregate result). Same `select()` gather underneath, though — see the
> [pure-filter-gather ticket](../../notes/tickets/pure-filter-gather.md), which narrows that gather
> in the scan copy.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor | :32-40 | `sirius_physical_operator(FILTER, types, cardinality)` + stores the single `expression`. **Asserts `expression != nullptr`** (:39) — FILTER always has exactly one predicate. |
| `execute` | :42-62 | `dynamic_cast` input to `pipelineable_operator_data` (:46), grab `get_read_only_batches()` (:47), build a fresh `gpu_expression_executor(*expression, current_device_resource, stream)` (:49-50), then **per batch** call `select(get_table_view())` (:56) and `make_data_batch` the result (:58-59). Returns a new `pipelineable_operator_data` (:61). |

The executor is constructed **per `execute()` call** (per task), bound to the task's stream and the
current device resource — so each task filters on its own GPU/stream. No shared state across tasks;
batches are processed independently, 1:1.

## The one idea: FILTER == `select()` == fused mask + gather

`gpu_expression_executor::select(table_view)` is a single-BOOLEAN-expression call that (a) evaluates
the predicate into one boolean mask column (fused AST kernel), then (b) `apply_boolean_mask`
**gathers** the surviving rows across *all* input columns. That gather is the expensive,
cardinality-changing step. Contrast the sibling PROJECTION, which uses `execute()` (N expressions →
N output columns, no row-count change). This `select()` vs `execute()` split is the crux of the
[fused-kernels note §4](../../notes/expressions/fused-kernels-and-breakers.md) and the pure-filter
gather ticket.

> **Stale header comment.** `sirius_physical_filter.hpp:27-29` says the filter "does not physically
> change the data, it only adds a selection vector to the chunk." That's an inherited DuckDB-ism —
> the GPU impl does a **real gather** (`apply_boolean_mask`), materializing new compacted columns,
> not a lazy selection vector. Read the `.cpp`, not that comment.

## Types fundamental to *this* file

- **`sirius::ast::node`** *(Sirius)* — the filter predicate, built by `from_duckdb` at plan time.
  Owned by the operator (`std::unique_ptr<sirius::ast::node> expression`, hpp :40).
- **`gpu_expression_executor`** *(Sirius)* — the evaluation engine; FILTER uses its single-expr
  `select()` entry point. See [its map](../expression_executor/gpu_expression_executor.md).
- **`pipelineable_operator_data`** *(Sirius)* — the batch carrier, both in (read-only batches) and
  out (freshly filtered batches).
- **`cucascade::data_batch` / `gpu_table_representation`** *(cuCascade)* — the GPU table wrapper;
  `batch.get_data()->cast<gpu_table_representation>().get_table_view()` is the input `cudf::table_view`,
  `make_data_batch(...)` wraps the output.

## Takeaway

`sirius_physical_filter` is a stateless, per-batch pass that maps each input batch through
`gpu_expression_executor::select()` (fused boolean mask + `apply_boolean_mask` gather) and rewraps
the survivors. Its whole substance is "which executor entry point" — `select()` (single predicate,
row-reducing gather) vs the projection's `execute()`. On the scan path the equivalent lives fused
inside the ingestible's `post_filter_and_project`; this operator is the non-pushed-down version.
Read with [`sirius_physical_projection`](sirius_physical_projection.md) and
[`gpu_expression_executor`](../expression_executor/gpu_expression_executor.md).
