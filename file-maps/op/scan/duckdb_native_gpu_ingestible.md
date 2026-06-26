# `op/scan/duckdb_native_gpu_ingestible.cpp` → Scan Map

One of the `gpu_ingestible` implementations introduced by the `#871` scan unification (see the
overview in [`duckdb_scan_executor.md`](duckdb_scan_executor.md)). ~329 lines. This map exists
because its **`post_filter_and_project`** is the *real* edit target of the scan
FILTER→PROJECTION gather ticket (see
[`../../../notes/expressions/fused-kernels-and-breakers.md`](../../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-25.

## Where this sits

A `gpu_ingestible` is the per-source object the unified `sirius_gpu_scan_operator` drives to
turn a table source into GPU batches: it provides **splits** (row-group ranges) and a
**post-read filter+projection** pass. `duckdb_native_gpu_ingestible` is the variant for the
duckdb-native base-table `seq_scan` path (siblings: `parquet_gpu_ingestible`,
`pinned_table_gpu_ingestible` — all three carry a `post_filter_and_project`). This is the **live**
home of the scan filter+project post-`#871`; the older
[`../sirius_physical_table_scan.md`](../sirius_physical_table_scan.md) `execute()` is a bypassed
copy of the same logic.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor | :84-166 | Validates the plan; **pre-builds the filter expression** (`convert_table_filters_to_expression`, :145) using an `emission_order_map` keyed off `projection_ids`; computes `_output_arity = output_types.size()` and `_projection_required` (:154-157). |
| `has_more_splits` / `next_split_provider` | :173-204 | Split-provider interface: hand out row-group ranges to worker threads. |
| `make_batch` | :209-231 | Packs a coalesced range into a work order; sets `pf->apply_filter` and **`pf->output_arity`** (:224) — the keep-set, as a count. |
| `consume_next_input` | :233-… | Coalesces parsed ranges into cap-sized batches. |
| **`post_filter_and_project`** | :299-327 | The ticket target. `:313` `exec.select(src->view())` = **gather (materializes all columns)**; `:316-324` `release()` + keep first `output_arity` columns = **project-back (metadata-only)**. |

The keep-set arrives here positionally as `output_arity`: columns `[0, output_arity)` are
outputs, `[output_arity, end)` are exactly the pure-filter columns appended by the planner
([`../../planner/sirius_plan_get.md`](../../planner/sirius_plan_get.md)). The waste: `:313`
produces the pure-filter columns; `:320` drops them. The fix is to slice `src->view()` to
`output_arity` columns **before** `apply_boolean_mask`.

## Types fundamental to *this* file

- **`io::post_filter_and_projection_info`** *(Sirius)* — the per-batch work order; the
  `duckdb_native` subclass carries `apply_filter` + `output_arity` (header :137-138).
- **`gpu_expression_executor`** *(Sirius)* — runs the pre-built filter; `select()` does the
  gather. Same engine as FILTER/PROJECTION
  ([`../../expression_executor/gpu_expression_executor.md`](../../expression_executor/gpu_expression_executor.md)).
- **`scan_plan`** *(Sirius)* — the read/output layout consumed during decode; see
  [`scan_plan.md`](scan_plan.md) for the longhand keep-vs-pure-filter classification.

## Takeaway

The live scan filter+project is `post_filter_and_project` here (and its two sibling ingestibles);
the `sirius_physical_table_scan::execute()` copy is bypassed post-`#871`. It already holds the
keep-set (here as `output_arity`); the ticket just applies it one step earlier. Read alongside
[`scan_plan.md`](scan_plan.md) (where output vs pure-filter is decided in longhand) and
[`../sirius_physical_table_scan.md`](../sirius_physical_table_scan.md) (the bypassed copy of the
same logic).
