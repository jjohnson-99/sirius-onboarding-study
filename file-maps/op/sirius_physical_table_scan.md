# `op/sirius_physical_table_scan.cpp` → Operator Map

The generic `TABLE_SCAN` operator (~249 lines). This map covers its **`execute()` that applies
the filter and projects** — but read the warning first: **that `execute()` is pre-`#871` and no
longer runs.** Post-unification the converter replaces the `TABLE_SCAN` source with a
`sirius_gpu_scan_operator` + `gpu_ingestible`, reading this operator only for *config*. The live
filter/project is in the ingestible's `post_filter_and_project`
([`scan/duckdb_native_gpu_ingestible.md`](scan/duckdb_native_gpu_ingestible.md)). This file is
still worth reading as a **self-contained statement** of the same gather-then-trim the ticket
targets (see [`../../notes/expressions/fused-kernels-and-breakers.md`](../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-25.

## Where this sits

Built by [`../planner/sirius_plan_get.md`](../planner/sirius_plan_get.md) from a `LogicalGet`.
It holds the DuckDB scan recipe (`column_ids`, `projection_ids`, `table_filters`,
`returned_types`) **plus** output `types`. Post-`#871` its main role is to **carry that recipe**:
the converter's `split_table_scan_source` (`sirius_pipeline_converter.cpp:930`) reads these fields
to build a `gpu_ingestible`'s `table_info`, inserts a `sirius_gpu_scan_operator` as the pipeline
source, and **drops this operator** — so its `execute()` below does not run on the live path.

The `execute()` reaches *down* into the expression executor
([`../expression_executor/gpu_expression_executor.md`](../expression_executor/gpu_expression_executor.md))
for the filter — the same `select()` FILTER uses — and is a faithful copy of the live ingestible
logic, which is why it's useful to read even though it's bypassed.

## `execute()` — the functions that matter

| Step | Lines | What |
|---|---|---|
| Passthrough shortcut | :117-119 | Parquet pipelines already filtered/projected upstream → return input as-is. |
| Coalesce small batches | :125-152 | `cudf::concatenate` many small coalesced batches into one for fewer kernel launches. |
| Build filter AST | :158-162 | `convert_table_filters_to_expression(table_filters, …)` → `sirius::ast::node`. |
| **FILTER (the gather)** | :164-170 | `gpu_expression_executor.select(table_view)` (:167) — computes the boolean mask then `apply_boolean_mask`, **materializing every column**, pure-filter ones included. |
| **PROJECT BACK (positional trim)** | :177-241 | `expected_output_columns = types.size()` (:177); if the batch is wider, keep `projection_ids[0 .. types.size())` (:223) — the original output columns by the planner's outputs-first convention. |

The waste pattern the ticket targets, in miniature: `:167` gathers pure-filter columns that
`:223` drops one step later. The trim is done *positionally* via `projection_ids` + `types.size()`.
The **live** version of this exact shape — the one that actually runs and that the ticket edits —
is `output_arity` in [`scan/duckdb_native_gpu_ingestible.md`](scan/duckdb_native_gpu_ingestible.md).
(Note `:117-119`'s `passthrough` early-return is dead: `passthrough` is never set `true`, and its
comment references the `#871`-deleted `parquet_scan_task`.)

## Types fundamental to *this* file

- **`gpu_expression_executor`** *(Sirius)* — the filter engine; `select()` = mask + gather.
- **`pipelineable_operator_data` / `cucascade::data_batch`** *(Sirius/cucascade)* — the
  read-only-locked GPU batch in/out; see the base [`sirius_physical_operator.md`](sirius_physical_operator.md).
- **`build_batch_column_map(projection_ids, column_ids.size())`** *(Sirius)* — maps a
  `projection_ids` entry to its physical batch column position (used by both the filter builder
  and the trim).

## Takeaway

`TABLE_SCAN` post-`#871` is a **config carrier**: its fields drive the GPU scan operator the
converter inserts, and its own filter/project `execute()` is a now-bypassed copy of that logic.
Read it as the clearest single-file statement of the gather-then-trim the ticket targets, then go
to the **live** site — [`scan/duckdb_native_gpu_ingestible.md`](scan/duckdb_native_gpu_ingestible.md)
(`post_filter_and_project`), where the ticket's fix actually lands (fold the keep-set into the
gather so pure-filter columns are never materialized).
