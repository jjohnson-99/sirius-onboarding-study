# `op/sirius_physical_projection.cpp` → Operator Map

The standalone **PROJECTION** operator (`SiriusPhysicalOperatorType::PROJECTION`): evaluate a
**list** of expressions, producing one output column each. It is the thinnest possible wrapper
around [`gpu_expression_executor::execute()`](../expression_executor/gpu_expression_executor.md) —
construct the executor from the `select_list`, run `execute()` per input batch, wrap the resulting
columns back into batches. Paired with [`sirius_physical_filter`](sirius_physical_filter.md) (the
`select()` sibling); the two are near-identical in shape and differ only in the executor entry point.

> Paths relative to the Sirius repo root; study-note links relative to this file. Lines as of
> 2026-07-01 — re-grep if the file moved. Tiny file (~69 lines); flat operator, no control-flow
> chain, so no call-tree (methods table below).

## Where this sits

A **regular streaming intermediate operator**: one input batch → one output batch, stateless, no
buffering. The plan generator builds it from DuckDB's `LOGICAL_PROJECTION` (see
[`../planner/plan-builders.md`](../planner/plan-builders.md)); at runtime a `gpu_pipeline_task`
calls its `execute()` in the pipeline's operator loop.

```
pipelineable_operator_data (input batches)
   └─ execute(): for each batch
        gpu_expression_executor(select_list).execute(table_view)   :59   N exprs → N columns
        make_data_batch(projected_table, mem_space, stream)        :62
   └─▶ pipelineable_operator_data (projected batches)
```

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor | :32-40 | `sirius_physical_operator(PROJECTION, types, cardinality)` + stores the `select_list` (a **vector** of AST nodes — one per output column). No non-null assert (unlike FILTER's single expression). |
| `execute` | :42-65 | `dynamic_cast` input to `pipelineable_operator_data` (:46), grab `get_read_only_batches()` (:47), build a fresh `gpu_expression_executor(select_list, current_device_resource, stream)` (:52-53), then **per batch** call `execute(get_table_view())` (:59) and `make_data_batch` the result (:61-62). Returns a new `pipelineable_operator_data` (:64). Carries a **TODO** (:49-51) to pick the execution strategy from statistics — [issue #636](https://github.com/sirius-db/sirius/issues/636). |

Like FILTER, the executor is built **per `execute()` call** (per task), bound to the task's stream
and current device resource; batches are processed independently, 1:1.

## The one idea: PROJECTION == `execute()` == N expressions → N columns

`gpu_expression_executor::execute(table_view)` evaluates the whole `select_list` and returns a table
with **one column per expression** — and the row count is unchanged (no gather). This is the key
contrast with FILTER's `select()` (single boolean predicate, row-reducing `apply_boolean_mask`
gather). The `select()` vs `execute()` distinction is the two-entry-point idea in the
[fused-kernels note §4](../../notes/expressions/fused-kernels-and-breakers.md).

> **Projection here is *compute*, not just column selection.** The `select_list` holds arbitrary
> expressions (`a + b`, casts, functions) — projection can synthesize new columns, not merely pick
> a subset. That's different from the *scan's* projection (`output_arity` / `output_layout`), which
> is a pure column **drop** of pure-filter columns. Don't conflate "the PROJECTION operator" with
> "the scan dropping pure-filter columns" — the latter is folded into the ingestible, not this
> operator (see [pure-filter-gather ticket](../../notes/tickets/pure-filter-gather.md)).

## Types fundamental to *this* file

- **`sirius::ast::node`** *(Sirius)* — each projected expression, built by `from_duckdb` at plan
  time. Owned as a vector (`duckdb::vector<std::unique_ptr<sirius::ast::node>> select_list`, hpp :39).
- **`gpu_expression_executor`** *(Sirius)* — the evaluation engine; PROJECTION uses its
  expression-list `execute()` entry point. See [its map](../expression_executor/gpu_expression_executor.md).
- **`pipelineable_operator_data`** *(Sirius)* — the batch carrier, in (read-only batches) and out
  (freshly projected batches).
- **`cucascade::data_batch` / `gpu_table_representation`** *(cuCascade)* — the GPU table wrapper;
  provides the input `cudf::table_view` and receives the output via `make_data_batch(...)`.

## Takeaway

`sirius_physical_projection` is a stateless, per-batch pass that maps each input batch through
`gpu_expression_executor::execute()` (evaluate the `select_list`, one output column per expression,
row count preserved) and rewraps the result. It is structurally identical to
[`sirius_physical_filter`](sirius_physical_filter.md); the only real difference is `execute()`
(N columns, no gather) vs `select()` (one predicate, row-reducing gather). Read with the filter map
and [`gpu_expression_executor`](../expression_executor/gpu_expression_executor.md).
