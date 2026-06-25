# `op/sirius_physical_table_scan.cpp` → Operator Map

The generic `TABLE_SCAN` operator (~249 lines). The `DUCKDB_SCAN` map
([`sirius_physical_duckdb_scan.md`](sirius_physical_duckdb_scan.md)) is built *from* this; this
map covers the part that one doesn't: the **`execute()` that applies the filter and projects**.
It is the DUCKDB-path site of the scan FILTER→PROJECTION gather ticket (see
[`../../notes/expressions/fused-kernels-and-breakers.md`](../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-25.

## Where this sits

Built by [`../planner/sirius_plan_get.md`](../planner/sirius_plan_get.md) from a `LogicalGet`.
It holds the DuckDB scan recipe (`column_ids`, `projection_ids`, `table_filters`,
`returned_types`) **plus** output `types`. Unlike `sirius_physical_duckdb_scan` (which has no
`execute()` — scanning is driven by the scan executor), this class **does** have an `execute()`
that runs on each GPU batch after it lands, and that's where the filter + project-back happen.

It reaches *down* into the expression executor
([`../expression_executor/gpu_expression_executor.md`](../expression_executor/gpu_expression_executor.md))
for the filter — the same `select()` FILTER uses.

## `execute()` — the functions that matter

| Step | Lines | What |
|---|---|---|
| Passthrough shortcut | :117-119 | Parquet pipelines already filtered/projected upstream → return input as-is. |
| Coalesce small batches | :125-152 | `cudf::concatenate` many small coalesced batches into one for fewer kernel launches. |
| Build filter AST | :158-162 | `convert_table_filters_to_expression(table_filters, …)` → `sirius::ast::node`. |
| **FILTER (the gather)** | :164-170 | `gpu_expression_executor.select(table_view)` (:167) — computes the boolean mask then `apply_boolean_mask`, **materializing every column**, pure-filter ones included. |
| **PROJECT BACK (positional trim)** | :177-241 | `expected_output_columns = types.size()` (:177); if the batch is wider, keep `projection_ids[0 .. types.size())` (:223) — the original output columns by the planner's outputs-first convention. |

The waste the ticket targets: `:167` gathers pure-filter columns that `:223` drops one step
later. The trim is done *positionally* via `projection_ids` + `types.size()` — it never reads
`original_projection_ids`. (The GPU-native scan does the identical thing via `output_arity`; see
[`scan/duckdb_native_gpu_ingestible.md`](scan/duckdb_native_gpu_ingestible.md).)

## Types fundamental to *this* file

- **`gpu_expression_executor`** *(Sirius)* — the filter engine; `select()` = mask + gather.
- **`pipelineable_operator_data` / `cucascade::data_batch`** *(Sirius/cucascade)* — the
  read-only-locked GPU batch in/out; see the base [`sirius_physical_operator.md`](sirius_physical_operator.md).
- **`build_batch_column_map(projection_ids, column_ids.size())`** *(Sirius)* — maps a
  `projection_ids` entry to its physical batch column position (used by both the filter builder
  and the trim).

## Takeaway

`TABLE_SCAN` is the one scan operator that filters+projects in its own `execute()`. The
project-back is real and correct; the ticket's fix is to fold the keep-set (first `types.size()`
columns) *into* the `:167` gather so it never materializes pure-filter columns — making the
`:177-241` trim block largely unnecessary. Sibling site:
[`scan/duckdb_native_gpu_ingestible.md`](scan/duckdb_native_gpu_ingestible.md).
