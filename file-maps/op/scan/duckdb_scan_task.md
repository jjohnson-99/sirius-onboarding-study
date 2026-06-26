# `op/scan/duckdb_scan_task.cpp` → Scan-Task Map

The runnable **unit of work** for the *DuckDB-table-function* scan path: it pulls DuckDB
`DataChunk`s off a CPU table function and packs them column-by-column into a pinned **host**
allocation, emits a `data_batch`, and self-reschedules a continuation task until the source
drains. ~638 lines (impl) + a header that defines its three state classes.

> ## ⚠️ VESTIGIAL post-`#871` — not produced by the live converter
>
> This is the **CPU-staging** scan path, driven by the `sirius_physical_duckdb_scan` (`DUCKDB_SCAN`)
> source. **Nothing in the live pipeline builds that source anymore.** It is constructed only by
> `construct_sirius_specific_operator` (`sirius_engine.cpp:231` — **zero callers**; converter
> free-fn `sirius_pipeline_converter.cpp:85` — called only with agg/distinct merge ops at
> `:653/:670/:906`, **never with a `TABLE_SCAN`**). The live scan rewriter
> `split_table_scan_source` (`:360`) turns **every** `TABLE_SCAN` — parquet *and* `seq_scan` —
> into a `GPU_SCAN` (`sirius_gpu_scan_operator` + `gpu_ingestible`). `duckdb_scan_task_global_state`
> (the entry to this family) is never constructed from a live path, and `get_scan_executor()` is
> called only from inside this dead family (`duckdb_scan_task.cpp:51`).
>
> Same status as [`../sirius_physical_table_scan.md`](../sirius_physical_table_scan.md)'s
> `execute()`: pre-`#871` code that still compiles but is bypassed. Read it to understand the
> *old* CPU-staging design, not the current control flow. The map below describes that code
> faithfully; just don't expect it to run.

This was the **CPU-staging** scan family — distinct from the GPU-native one. Keep the names
straight (the collision below is why this map exists):

| Family | Source operator | Driver | Status |
|---|---|---|---|
| **DuckDB-native (CPU staging)** | `sirius_physical_duckdb_scan` (`DUCKDB_SCAN`) | `duckdb_scan_executor` thread pool | **vestigial** — `duckdb_scan_task` (this file) is its task |
| **GPU-native decode** | `sirius_gpu_scan_operator` (`GPU_SCAN`) | `scan_manager` + `gpu_ingestible` | **live** — see [`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md) / [`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md) |

> Note the near-collision: *`duckdb_native_gpu_ingestible`* (GPU family, **live**) vs
> *`duckdb_scan_task`* (this, CPU family, **vestigial**). "DuckDB" appears in both because both
> ultimately read a DuckDB-registered table; the split is **who decodes** — DuckDB's CPU table
> function here, cuDF on-GPU there. The *live* path is always the GPU one today.

> Paths relative to the Sirius repo root. Lines as of 2026-06-26 — re-grep if the file moved.

## Where this sits

`sirius_physical_duckdb_scan` is a pipeline **source**. Its executor
([`duckdb_scan_executor.md`](duckdb_scan_executor.md)) owns a host thread pool and dispatches
`duckdb_scan_task`s; each task runs the DuckDB table function, stages rows into host memory, and
publishes batches to the `shared_data_repository` that downstream operators consume (the
host→GPU upload happens later, in the consuming operator/pipeline). The task self-replicates: as
long as its local table-function state isn't drained, it schedules the *next* task carrying the
moved-along `LocalTableFunctionState` so the scan resumes where it left off.

```
duckdb_scan_executor (thread pool)
   └─ schedules duckdb_scan_task
        execute()                                   :531
          compute_task()                            :555
            loop get_next_chunk → chunk_fits → process_chunk   :567
            if not drained → schedule continuation task        :586
            make_data_batch (host pinned)           :444 (l_state)
          publish_output → data_repo->add_data_batch :615
```

## Three classes (defined in the header)

| Class | Role |
|---|---|
| `duckdb_scan_task_global_state` | Per-source shared state: holds the DuckDB **global** table-function state, the task scheduler, `_max_threads` (= executor thread count), and the drain bookkeeping (`_active_local_states`, `_source_drained`). Doubles as a `duckdb::GlobalSourceState`. |
| `duckdb_scan_task_local_state` | Per-task state: the pinned host `_allocation`, one `column_builder` per column, the DuckDB **local** table-function state, and the reusable `_chunk`. Owns `make_data_batch`. |
| `duckdb_scan_task` | The `sirius_pipeline_itask` itself: `execute` / `compute_task` / `publish_output`. |

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| `global_state` ctor | :44-80 | Inits the DuckDB **global** TF state — **passes `nullptr` for filters** (:63): filters are applied by Sirius on GPU, not by the table function. Rejects `in_out_function` (:69) and `dynamic_filters` (:76). |
| `set_source_drained` / `decrement_local_states` | hpp :101 / :126 | Drain accounting: when the last active local state finishes, mark the source drained and flip the operator's `exhausted` flag. |
| `column_builder` ctor / `initialize_accessors` | :85 / :92 | A per-column host-buffer writer. VARCHAR gets offset+data+mask sub-buffers; fixed-width gets data+mask. Computes byte layout into the shared allocation. |
| `sufficient_space_for_column` | :121 | VARCHAR-only pre-check: does the chunk's string bytes still fit in the pre-sized data buffer? |
| `process_mask_for_column` | :137 | Copies the DuckDB validity bitmap into the column's null mask, handling byte-aligned and bit-unaligned cases + all-valid fast path. |
| `process_column` | :221 | Copies a flattened DuckDB `Vector`'s data into the host buffer (string copy for VARCHAR, `memcpy` for fixed-width), then the mask. |
| `make_column_metadata` | :260 | Builds the cuDF `column_metadata` (STRING with offsets child, or fixed-width) describing the staged buffer so it can be read as a cuDF column. |
| `local_state` ctor | :315 | Requests a **HOST** memory reservation sized to `approximate_batch_size` (:336), allocates a `multiple_blocks_allocation` (:349), estimates rows/batch, builds the column builders, inits the DuckDB **local** TF state (reusing a moved-in one for continuation). |
| `estimate_rows_per_batch` | :371 | Computes rows/batch from per-column widths (real fixed widths + `default_varchar_size`), floored at `STANDARD_VECTOR_SIZE`. |
| `initialize_builders` | :404 | Lays out each column's sub-buffers at 8-byte-aligned offsets within the allocation. |
| `make_data_batch` | :444 | Wraps the filled allocation as a `host_table_allocation` → `host_data_representation` → `data_batch`. |
| `get_next_chunk` | :482 | Calls the DuckDB table function for one `DataChunk`; on empty, marks the local state drained (`decrement_local_states`). |
| `chunk_fits` | :504 | Pre-flight VARCHAR space check across the chunk before processing. |
| `process_chunk` | :519 | Flatten + `process_column` each column; advance `_row_offset`. |
| `compute_task` | :555 | The body: init `_chunk` with `scanned_types`, loop chunks until the batch fills, **schedule a continuation task** (:586-603) if the source isn't drained, then return the batch (or empty). |
| `execute` | :531 | `compute_task` + record memory metrics (output bytes as the peak proxy) + `publish_output`. |
| `publish_output` | :615 | `data_repo->add_data_batch` for each output batch. |
| `get_estimated_reservation_size_info` | :623 | Reservation estimate from the memory history keyed on `input_basis` (= batch size). |

## Types fundamental to *this* file

- **`sirius_physical_duckdb_scan`** *(Sirius)* — the source operator this task drives; carries
  `function` (the DuckDB `TableFunction`), `bind_data`, `column_ids`, `projection_ids`,
  `scanned_types`. See [`../sirius_physical_duckdb_scan.md`](../sirius_physical_duckdb_scan.md).
- **`duckdb::TableFunction` / `TableFunctionInput` / Global+LocalTableFunctionState`** *(DuckDB)*
  — the CPU scan engine being pumped. Filters are deliberately withheld (`nullptr`).
- **`column_builder`** *(Sirius, this file)* — the per-column host-staging writer + its
  allocation accessors (`multiple_blocks_allocation_accessor`).
- **`cucascade::memory::host_table_allocation` / `host_data_representation` / `data_batch`**
  *(cucascade)* — the host-resident batch produced; `column_metadata` describes its cuDF layout.
- **`memory_reservation` / `fixed_size_host_memory_resource`** *(cucascade)* — the HOST-tier
  reservation + block allocator backing `_allocation`.
- **`sirius_pipeline_itask`** *(Sirius)* — the task base; `execute`/`compute_task`/`publish_output`
  contract shared with other pipeline tasks.

## How it fits with the rest of the scan stack

- **Executor** ([`duckdb_scan_executor.md`](duckdb_scan_executor.md)): owns the thread pool and
  schedules these tasks; this file is the per-task logic the executor runs.
- **Source operator** ([`../sirius_physical_duckdb_scan.md`](../sirius_physical_duckdb_scan.md)):
  the `DUCKDB_SCAN` that this task reads config from (`function`, `scanned_types`, ids).
- **GPU-native sibling** ([`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md)):
  the **live** scan path that *superseded* this one — contrast it to keep "who decodes" straight
  (DuckDB CPU here, cuDF on-GPU there). The filter/project ticket lives in that family, **not**
  this one (this path withholds filters from DuckDB and applies them downstream).

## Takeaway

`duckdb_scan_task` *was* the CPU-staging scan worker: pump a DuckDB table function, pack chunks
into a pinned host batch, publish, and re-queue a continuation until drained. **Post-`#871` it is
vestigial** — the converter no longer builds the `DUCKDB_SCAN` source that drives it; every
`TABLE_SCAN` becomes a `GPU_SCAN`. Read this map to understand the old CPU-staging design (and as
the foil for "duckdb-native means GPU here"), not the live control flow. See
[`duckdb_scan_executor.md`](duckdb_scan_executor.md) (its old driver) and
[`../sirius_physical_duckdb_scan.md`](../sirius_physical_duckdb_scan.md) (its source operator).
