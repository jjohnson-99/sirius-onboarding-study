# `op/scan/scan_plan.cpp` → Scan Map

The scan's read/output layout planner (~256 cpp + 193 hpp). This map exists because
`build_scan_plan` is where **output vs pure-filter columns are classified in longhand** — the
richer form of the positional convention used by the gather ticket (see
[`../../../notes/expressions/fused-kernels-and-breakers.md`](../../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-25.

## Where this sits

A pure planning/bookkeeping struct (no GPU work): given `column_ids`, `projection_ids`, names,
types and hive-partition indices, it precomputes everything the GPU scan needs to (a) decide
*which* columns to read, (b) inject partition columns, and (c) **assemble the final output
layout**. It's consumed by the `gpu_ingestible` decoders
([`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md) and siblings).

## The classification (longhand)

`build_scan_plan` (`:157-…`) walks `projection_ids` in output-first order and tags each column:

| Mechanism | Lines | What |
|---|---|---|
| `needs_reader_projection` | :178 | True when the planner pruned/reordered columns or partition columns must be stripped. |
| `is_output` flag | :150-153, :233 | `is_output_position(i, output_types_size)` — the **first `output_types_size` entries are outputs**; the rest are pure-filter ("must be read but not emitted", comment :180-184). |
| `data_columns` vs `output_layout` | :210-222 | A data column is *always* added to `data_columns` (even filter-only — needed for filter eval), but only output columns get an `output_layout` entry. **That split is the keep-vs-pure-filter classification.** |
| `assemble_scan_output` | hpp :165 | Materializes the final table per `output_layout`, dropping data columns not referenced by it (i.e., pure-filter columns). |

This is the same fact `sirius_plan_get.cpp` encodes positionally (outputs first), just made
explicit: `output_layout` = keep, `data_columns \ output_layout` = pure-filter.

## Types fundamental to *this* file

- **`scan_plan`** *(this file; hpp :59)* — the struct: `data_columns` (:82), `partition_columns`,
  `output_layout` (:88), `needs_reader_projection` (:105), and the `C→D` position map.
- **`scan_plan::output_entry`** *(hpp :74)* — tags an output slot as `DATA` or `PARTITION` plus
  an index into the matching vector.
- **`duckdb::ColumnIndex` / `HivePartitioningIndex`** *(DuckDB)* — read indices + partition
  columns injected from the file path.

## Takeaway

`scan_plan` is the authoritative, longhand statement of "which columns survive into the
pipeline" (`output_layout`) vs "which are read only to evaluate filters" (`data_columns`
minus that). The ticket's keep-set is exactly `output_layout`; the gather should be narrowed to
it. Read with [`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md), which
consumes this plan and is where the gather-then-drop actually runs.
