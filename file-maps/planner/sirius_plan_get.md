# `planner/sirius_plan_get.cpp` → Plan-Builder Map

The `LOGICAL_GET` builder. Listed as a one-liner in
[`plan-builders.md`](plan-builders.md) (row `sirius_plan_get.cpp`); this map expands it
because it is where **pure-filter columns get classified** — the planner-side root of the
scan FILTER→PROJECTION gather ticket (see
[`../../notes/expressions/fused-kernels-and-breakers.md`](../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-25 — re-grep if the file moved.

## Where this sits

`sirius_physical_plan_generator::create_plan(LogicalGet&)` is one of the per-node builders the
plan dispatcher calls (see [`sirius_physical_plan_generator.md`](sirius_physical_plan_generator.md)).
It turns a DuckDB `LogicalGet` (a table read, possibly with pushed-down projection and filters)
into a Sirius scan operator — `sirius_physical_table_scan`
([`../op/sirius_physical_table_scan.md`](../op/sirius_physical_table_scan.md)), refined later
into `DUCKDB_SCAN` or a `GPU_PARQUET_SCAN` depending on the table function.

Its single most important output for the ticket is **`projection_ids`**: it decides which read
columns are output vs read-only-for-filtering, and bakes that into the operator.

## The classification (the part the ticket cares about)

| Step | Lines | What |
|---|---|---|
| Remap table filters to scan-local column indices | `create_table_filter_set` :52-71 | `table_filters` keyed by storage index → keyed by position in `column_ids`. |
| **Snapshot `projection_ids` before augmenting** | :111 | `original_projection_ids` = the columns the query actually outputs (set by DuckDB projection pushdown). **Vestigial — never read again.** |
| **Force filter columns into the read set** | :117-133 | For each filter column not already an output, `projection_ids.push_back(entry.first)` (:131). After this, `projection_ids = [outputs…, pure-filter…]`. **The push at :131 *is* the pure-filter classification.** |
| Type-pushdown-unsupported fallback | :135-179 | Columns whose type the table function can't filter get a *separate* `sirius_physical_filter` stacked on top instead. |
| Non-pushdown path | :182-241 | Table function lacks projection pushdown: read all `returned_types`, then `push_projection` to `column_ids` order on top. |
| Pushdown path | :243-262 | Build `sirius_physical_table_scan` with augmented `op.projection_ids` (:249) + original output `types` (:244). The trim is **deferred to scan time** — done positionally by the GPU scan operator's ingestible (`output_arity`), since the converter replaces this `TABLE_SCAN` with that operator. |

The convention this establishes — **outputs first, pure-filter columns appended** — is what the
scan operator (and the GPU-native ingestible) rely on to trim later: keep the first
`types.size()` / `output_arity` columns. So:

```
pure_filter_columns  ==  projection_ids (after :131)  \  original_projection_ids (:111)
                     ==  projection_ids[ types.size() : ]   (positionally)
```

`original_projection_ids` is dead because that second, positional form carries the same
information — not because the trim is missing.

## Types fundamental to *this* file

- **`duckdb::LogicalGet`** *(DuckDB)* — the logical table read: carries `column_ids`,
  `projection_ids`, `table_filters`, `function` (the `TableFunction`), `returned_types`.
- **`duckdb::TableFilterSet`** *(DuckDB)* — pushed-down WHERE predicates, keyed by column.
- **`column_ids` vs `projection_ids`** — read set vs output-selection-into-read-set; see the
  vocabulary box in the note's §5.

## Takeaway

This builder is where "which columns are output" becomes a concrete, *positional* fact on the
scan operator. The ticket doesn't change the classification here — it's already correct — it
gives the classification a consumer earlier (the gather) instead of trimming *after*
materializing. Read next: [`../op/sirius_physical_table_scan.md`](../op/sirius_physical_table_scan.md)
(the config carrier this builds) → its live replacement
[`../op/scan/duckdb_native_gpu_ingestible.md`](../op/scan/duckdb_native_gpu_ingestible.md).
