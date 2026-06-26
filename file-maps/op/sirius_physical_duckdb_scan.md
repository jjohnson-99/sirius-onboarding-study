# `op/sirius_physical_duckdb_scan.cpp` → Operator Map (tiny)

Companion for Week 2, **Days 4–5** of [`onboarding-path.md`](../../onboarding-path.md).
Tagged **tiny** (~130 lines, almost all constructor). Read with the `DUCKDB_SCAN`
entry in `docs/super-sirius/operators.md`.

> ## ⚠️ VESTIGIAL post-`#871` — no longer the live `lineitem` source
>
> Re-verified 2026-06-26: this operator is **not produced by any live path**. The live scan
> rewriter `split_table_scan_source` (`converter:360`) turns **every** `TABLE_SCAN` — including
> `seq_scan` over a base table — into a `GPU_SCAN` (`sirius_gpu_scan_operator` +
> `duckdb_native_gpu_ingestible`), *not* a `DUCKDB_SCAN`. `sirius_physical_duckdb_scan` is built
> only by `construct_sirius_specific_operator` — engine member `sirius_engine.cpp:231` (**zero
> callers**) and converter free-fn `converter.cpp:85` (called only with agg/distinct merge ops at
> `:653/:670/:906`, **never `TABLE_SCAN`**). So for `SELECT … FROM lineitem …` today the source is
> a `GPU_SCAN`, and the actual reading runs through the
> [`duckdb_native_gpu_ingestible`](scan/duckdb_native_gpu_ingestible.md), not the
> [`duckdb_scan_executor`](scan/duckdb_scan_executor.md)/[`duckdb_scan_task`](scan/duckdb_scan_task.md)
> path this map points to. Read this map for the *old* design and the still-true "a scan operator
> only *describes* a scan" lesson — not the current control flow.

> Paths relative to the Sirius repo root. Lines as of 2026-06-10 (intro re-verified 2026-06-26).

## Where this sits (pre-`#871`)

In the **old** design, for `SELECT ... FROM lineitem ...` the plan generator emitted a
`TABLE_SCAN`; `construct_sirius_specific_operator`
([`../pipeline/sirius_pipeline_converter.md`](../pipeline/sirius_pipeline_converter.md),
converter.cpp:61) saw the table function was
`seq_scan` and built a **`sirius_physical_duckdb_scan`** — a sequential scan that
borrowed DuckDB's own execution engine to read the table, then handed rows to the GPU.
(A parquet file became `PARQUET_SCAN` instead.) It was the `operators[0]` source of the
main pipeline. **Post-`#871` that refinement is gone** — see the banner above.

## The key thing to notice: this file has no `execute()`

The `.cpp` is **only constructors**. There is no `execute()` here. That's the lesson:

> A scan operator is mostly a **data holder describing *what* to scan**; the *act* of
> scanning is driven elsewhere — by the **scan executor** (`duckdb_scan_executor`,
> Week 5 reading), not by an `execute()` on this class.

What the operator carries (header) is the DuckDB scan recipe: `function` (the
`TableFunction`), `bind_data`, `column_ids`, `projection_ids`, `table_filters`
(pushed-down predicates), `names`, `returned_types`. The constructor's only real work
(cpp 85–126) is computing `scanned_types` — the types of the columns DuckDB will
actually emit after projection — and sorting `projection_ids` so output column order
matches the parquet path's convention.

## The two overrides that matter

| Member | Where | Role |
|---|---|---|
| `bool is_source() const` → `true` | header 105 | Marks it a pipeline source (produces data, has no operator children). |
| `get_next_task_hint()` | header 52 | Custom readiness: returns `READY` until the atomic `exhausted` flag is set, then `nullopt`. So the task creator keeps making scan tasks until the table is drained. |
| `std::atomic<bool> exhausted` | header 102 | Set by the scan executor when the table function reports no more rows; this is the scan's "I'm done" signal that lets the pipeline finish. |

Contrast with the base `get_next_task_hint()` (which reasons about input ports):
DUCKDB_SCAN has no input ports — it's a leaf — so it overrides with a trivial
"ready until exhausted."

## Types fundamental to *this* file

- **`sirius_physical_table_scan`** *(Sirius)* — the generic `TABLE_SCAN` this is built
  from; the constructor at cpp 37 copies its fields. **Think:** the pre-refinement
  scan node.
- **`duckdb::TableFunction` / `FunctionData` (bind_data)** *(DuckDB)* — the table
  function and its bound state that the scan executor will actually invoke to read
  rows. **Think:** "the DuckDB recipe for reading this table," reused on the GPU side.
- **`duckdb::ColumnIndex` / `projection_ids` / `TableFilterSet`** *(DuckDB)* — which
  columns to read and which predicates were pushed into the scan. **Think:** the
  column/filter selection.

## Takeaway

Two things to retain: (1) a scan operator *describes* a scan; something else *runs* it — so
don't look for GPU work here. (In the live engine that runner is the
[`gpu_ingestible`](scan/duckdb_native_gpu_ingestible.md) driven by
[`sirius_gpu_scan_operator`](scan/sirius_gpu_scan_operator.md); in this vestigial operator's old
design it was the `duckdb_scan_executor`.) (2) Its `exhausted` flag is how a source signals
completion, the mirror of the limit's `_limit_exhausted`. Next: the aggregate, where the target
query's `SUM` actually gets computed.
