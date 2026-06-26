# `op/scan/parquet_gpu_ingestible.cpp` → Scan Map

The parquet `gpu_ingestible` implementation (`#871`). It is the **fresh-read** sibling of
[`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md): same `io::gpu_ingestible`
interface, parquet-specific decode. **Still live** — `make_gpu_ingestible` →
`parquet_ingestible_table_info::make_ingestible` builds it for any parquet source that isn't
served from the pinned cache. ~639 lines.

This map matters for the FILTER→PROJECTION gather ticket because the parquet path handles the
filter **differently** from duckdb-native: the gather lives inside `materialize_table`, and its
`post_filter_and_project` does **assembly only**. (See the walkthrough
[`../../../notes/walkthroughs/scan-pipeline-and-filter-project.md`](../../../notes/walkthroughs/scan-pipeline-and-filter-project.md)
and concept note [`../../../notes/expressions/fused-kernels-and-breakers.md`](../../../notes/expressions/fused-kernels-and-breakers.md) §5.)

> Paths relative to the Sirius repo root. Lines as of 2026-06-26 — re-grep if the file moved.

## Where this sits

A `gpu_ingestible` is the per-source object the unified [`sirius_gpu_scan_operator`](sirius_gpu_scan_operator.md)
drives. The [`scan_manager`](../../scan_manager/sirius_scan_manager.md) builds one, installs it,
and runs a `split_provider` that composes the **same** ingestible to emit splits. For parquet:

```
parquet_ingestible_table_info::make_ingestible  → parquet_gpu_ingestible    :103
   ctor: build scan_plan, coalesce filter expr, decompose files into batches :112
   has_more_splits / next_split_provider → run_batch                         :168 / :173
      run_batch: footer read · FLBA probe · row-group prune · prewarm ·
                 partition-by-bytes → scan_operator_input splits             :188
   materialize_table (per task): read_parquet + filter (pushdown OR select)  :530
   post_filter_and_project: assemble_scan_output ONLY                        :623
```

## Two filter paths — the key parquet nuance

The filter is **fully resolved inside `materialize_table`**, two ways:

1. **Reader-side pushdown** — when AST translation succeeds and the file isn't FLBA-decimal,
   `opts.set_filter(...)` (:573) hands the predicate to cuDF's parquet reader for row-group
   pruning + row filtering during decode. No separate gather.
2. **Post-decode fallback** — when pushdown is disabled (FLBA-decimal probe, :566) or AST
   translation failed, `gpu_expression_executor::select(input->view())` (:598) does the
   **gather** after decode. *This* is the parquet analogue of the ticket's gather.

So `post_filter_and_project` (:623) is **assembly only** (`assemble_scan_output` — hive-partition
injection + column reordering), emitted only when `needs_output_assembly(*_plan)` is true.
Contrast duckdb-native, where `post_filter_and_project` *is* the gather. **The ticket's "skip
materializing pure-filter columns before the gather" maps to `materialize_table:598` here, not to
`post_filter_and_project`.**

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| `make_ingestible` | :103-107 | `ingestible_table_info` factory hook → constructs the ingestible. |
| ctor | :112-161 | Builds the shared `scan_plan` (:129), coalesces the DuckDB filter expression dropping hive-partition-column filters (:139-147), and **pre-decomposes** the file list into per-task `file_batch`es of `max_file_processed` files each (:154-160). |
| `has_more_splits` / `next_split_provider` | :168 / :173 | Atomically claim the next batch index; return a callable that runs `run_batch`. |
| `run_batch` | :188-525 | Port of `parquet_split_provider::run_batch`. Builds shared `reader_options` (column projection :197), translates the filter + **FLBA-decimal pushdown probe** (:205-280), then per file: resolve datasource, read/cache footer metadata, select projected leaf chunks (tracking `pure_filter_chunk_indices`, :374-392), prune row groups by stats (:394-407), **scan-side chunk prewarm** into the prefetch cache (:414-464), and bin row groups into byte-budgeted `parquet_split_info` splits (`seal_current_file`/`flush`). |
| `materialize_table` | :530-618 | Per-task read: build cudf datasources from the slices, `read_parquet` onto the task's memory space (:579), apply the filter (pushdown engaged → reader did it; else `exec.select` gather :598), and **inline assembly** on the pushdown path so the operator can skip `post_filter_and_project` (returns `ROW_FILTERED_AND_PROJECTED`, :610-615). |
| `post_filter_and_project` | :623-637 | **Assembly only** — `assemble_scan_output` to the plan layout. Reached only when `materialize_table` returned an un-assembled, filtered table (non-pushdown fallback path). |

## Types fundamental to *this* file

- **`parquet_ingestible_table_info`** *(this file, hpp)* — parquet bind-data carrier +
  `make_ingestible` factory; parked on the scan operator until `prepare_for_query`. Carries
  `column_ids`, `projection_ids`, `names`, `table_filters`, `partition_indices`,
  `scan_output_arity`, `max_file_processed`.
- **`parquet_split_info : io::scan_info`** *(hpp)* — per-split scan descriptor: `rg_slices`,
  shared `reader_options`, shared `plan`, `disable_filter_pushdown`, `partition_values`,
  `needs_assembly`.
- **`parquet_post_filter_and_projection_info : io::post_filter_and_projection_info`** *(hpp)* —
  carries `partition_values` for assembly.
- **`scan_plan`** *(Sirius)* — read/output layout + pure-filter classification; shared via
  `shared_ptr<const>`. See [`scan_plan.md`](scan_plan.md).
- **`gpu_expression_translator` / `gpu_expression_executor`** *(Sirius)* — translate the DuckDB
  filter to a cuDF AST (pushdown) and, on miss, run `select` (the post-decode gather). Same
  engine as FILTER/PROJECTION ([`../../expression_executor/gpu_expression_executor.md`](../../expression_executor/gpu_expression_executor.md)).
- **`hybrid_scan_reader` / `cudf::io::read_parquet`** *(cuDF)* — footer parse + row-group decode.
- **`io::filter_state`** *(Sirius io)* — the return tag (`UNFILTERED` / `ROW_FILTERED` /
  `ROW_FILTERED_AND_PROJECTED`) that tells the operator whether to skip `post_filter_and_project`.

## How it fits with the rest of the scan stack

- **Operator** ([`sirius_gpu_scan_operator.md`](sirius_gpu_scan_operator.md)): drives this via
  `materialize_table` + conditional `post_filter_and_project`; the `filter_state` it returns
  decides the skip.
- **scan_manager** ([`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md)):
  builds it, runs the split_provider over it.
- **Pinned sibling** ([`pinned_table_gpu_ingestible.md`](pinned_table_gpu_ingestible.md)): the
  cache path that *replaces* this when a pinned entry matches the file paths.
- **duckdb-native sibling** ([`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md)):
  contrast — there `post_filter_and_project` *is* the gather; here the gather is in `materialize_table`.

## Takeaway

`parquet_gpu_ingestible` is the live parquet fresh-read source: it decomposes files into
byte-budgeted row-group splits, prunes/pushes-down filters at the reader, and decodes per task.
Its defining trait vs the other ingestibles is that the **filter is resolved inside
`materialize_table`** (reader pushdown or a post-decode `select`), leaving `post_filter_and_project`
to do only hive/column assembly. Read with [`scan_plan.md`](scan_plan.md) (the layout it consumes)
and the two sibling ingestible maps.
