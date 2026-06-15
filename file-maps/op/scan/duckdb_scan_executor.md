# `op/scan/duckdb_scan_executor.cpp` → Scan-Subsystem Map

Companion for **Week 5** of [`onboarding-path.md`](../../../onboarding-path.md) ("Scan &
data flow"), tagged **read** — the scan *executor* (the plan says read this + the executor
logic in `src/op/scan/`, and only *skim* the GPU parquet decoders in `src/cuda/scan/`).
Read with `docs/super-sirius/scan.md`; prime with
[`../../../reference/explainers/parquet-format.md`](../../../reference/explainers/parquet-format.md).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file (note the **three** `../` — this map is three levels deep, mirroring
> `src/op/scan/`). Lines accurate as of 2026-06-15 — re-grep.

## Scope note — what to read vs. skim

`docs/super-sirius/scan.md` is the largest doc and describes **four** scan paths
(DuckDB, legacy parquet, GPU parquet via the scan manager, Iceberg) plus a whole io_uring
I/O stack. **For Week 5, don't try to hold all of it.** The plan's target is the
**executor + scan-task flow** that brings table data in from the host. This map stays
there; the scan_manager / `sirius::io` / Iceberg machinery are pointers-only (read the doc
if a task takes you there).

> **Updated for pull `d9172de6` (2026-06-15) — the GPU scan paths were restructured.**
> `#871` unified the GPU scan operators behind a **`gpu_ingestible`** abstraction:
> `parquet_scan_task.cpp`, `sirius_gpu_parquet_scan_operator.cpp`, and
> `sirius_gpu_duckdb_native_scan_operator.cpp` were **deleted**, replaced by
> `sirius_gpu_scan_operator.cpp` + `{parquet,duckdb_native,pinned_table}_gpu_ingestible.cpp`
> (and `row_group_metadata.hpp`); the native-metadata walk was parallelized/typed (`#895`,
> `#936`). **This map's subject file — `duckdb_scan_executor.cpp` — did *not* change**, so the
> executor + DuckDB-scan-task content below is intact. But the "four scan paths" framing and
> the GPU-parquet pointers now refer to the unified operator; treat scan-path names as
> approximate until `scan.md` is refreshed. Full diff in
> [`../../../reports/post-pull-review-2026-06-15.md`](../../../reports/post-pull-review-2026-06-15.md).

## Where this sits

The scan subsystem is the **source** end of every query — Step 6 of the execution flow,
the thing that feeds the operators you've been reading. `duckdb_scan_executor` is the
**third executor** in the tier (alongside the per-GPU
[`../../pipeline/gpu_pipeline_executor.md`](../../pipeline/gpu_pipeline_executor.md) and the
downgrade executor): all three inherit `itask_executor`. The
[`../../creator/task_creator.md`](../../creator/task_creator.md) builds `duckdb_scan_task`s
and dispatches them *here*; this executor runs them on a host worker pool, applies caching,
publishes the result batch into a data repository, then asks the creator to schedule the
downstream consumers.

It's the **host-I/O** executor: scans read from DuckDB/files (CPU-side), build a
`host_data_representation`, and the GPU-conversion happens later (when a pipeline task
locks the batch onto a device — the per-task-device contract from
[`../../pipeline/gpu_pipeline_executor.md`](../../pipeline/gpu_pipeline_executor.md)).

## The key idea first: same executor shape, host flavor + a cache

`manager_loop()` (cpp, override of the `itask_executor` hook) is the familiar
reserve→pop→dispatch loop, specialized for scans:

```
manager_loop():
├─ kiosk/pool.reserve()                 wait for a worker slot
├─ task_queue.pop()  (try non-blocking, else submit_scan_request() then block)
├─ for parquet: acquire a HOST memory reservation
└─ dispatch to worker:
     ├─ stream = per-GPU stream pool .acquire()    (target-bound — see FIX-01 note below)
     ├─ get_scan_output(task, stream)              :hpp 177   ── runs scan + applies caching
     ├─ task->publish_output()                     store the batch into its data repository
     └─ task_creator->schedule(consumers)          kick off whoever consumes this scan
```

The two things that make the scan executor distinct from the GPU executor are **caching**
(`get_scan_output`) and **multi-GPU target selection** (`select_target_gpu`).

## Caching — four levels (`get_scan_output`, `op/scan/config.hpp`)

The biggest reason this executor is interesting: re-running the same query can skip work.
`get_scan_output()` branches on `_cache_level`:

| `cache_level` | What's cached | Re-run cost |
|---|---|---|
| `NONE` *(default)* | nothing | full scan every time |
| `PARQUET` | raw compressed parquet bytes (host) | re-decompress each run; smallest footprint |
| `TABLE_HOST` | decoded table (host) | skip decompression; medium memory |
| `TABLE_GPU` | decoded table (GPU) | fastest — no data movement; highest memory |

Cache mechanics: `cache_scan_results_for_query(query)` (cpp 121) returns **true on a cache
hit** (same query text → preload mode); `prepare_cache_for_scan_operators` (cpp 153)
creates per-operator-id entries (CACHE mode) or verifies them present (PRELOAD mode).
Clone-on-serve rule: `TABLE_GPU` returns the original (GPU-resident, zero-copy); other
levels clone to avoid sharing cache data; parquet uses a refcount `shallow_clone()`.

> **Honesty / footgun.** Cache keying is by **query text** and by **operator id**, and the
> warm path depends on tasks landing on the *same* device across re-runs (this is exactly
> why the scheduler's device assignment had to become reproducible — see the drift note in
> [`../../pipeline/task_scheduler.md`](../../pipeline/task_scheduler.md)). Non-cacheable
> statements (`SET`, `CREATE VIEW`) are filtered by `is_cacheable_query_text` (cpp 60) so
> they don't invalidate the cache.

## Multi-GPU target selection (`select_target_gpu`, cpp 197)

Scan batches are distributed across GPUs **proportional to available GPU memory**, falling
back to round-robin (`_scan_round_robin`) when all are at capacity. A **sticky
batch→GPU affinity** map (`_batch_gpu_affinity`, hpp 208–217) records the choice so the
same batch re-targets the same GPU on warm runs (keeps `TABLE_GPU` cache hits valid). The
header carries a load-bearing **FIX-01** comment (hpp 191–198): per-GPU stream pools
replaced a single GPU-0-bound pool that caused a post-ship `cudaErrorInvalidValue` — a
concrete example of "a CUDA stream is bound to the device whose context created it."

## The scan-task flow (where data actually enters) — `duckdb_scan_task`

The executor runs tasks; the task does the reading (`src/op/scan/duckdb_scan_task.cpp`):
1. `get_next_chunk()` — pull a `DataChunk` from the DuckDB table function;
2. `chunk_fits()` — does it fit the pre-allocated column builders?
3. `process_chunk()` — write columns into **column builders** (fixed-width: data+validity;
   VARCHAR: offsets+data+validity; 8-byte aligned);
4. repeat until the target batch size or the source drains;
5. build a `host_data_representation`; if not done, **self-schedule** the next scan task.

This is the counterpart to the DUCKDB_SCAN operator you read in Week 2
([`../sirius_physical_duckdb_scan.md`](../sirius_physical_duckdb_scan.md)) — the operator
was a *descriptor*; the **execution lives here**.

## Methods (hpp)

| Member (signature) | Line | Role |
|---|---|---|
| `duckdb_scan_executor(config, mem_mgr, task_request_publisher)` | hpp 72 | host worker pool + memory mgr for host reservations. |
| `void manager_loop()` *(override)* | hpp 159 | the reserve→pop→dispatch loop above. |
| `get_scan_output(task, stream)` | hpp 177 | run the scan, apply caching. |
| `int select_target_gpu()` | hpp 175 | memory-proportional GPU choice for the batch. |
| `bool cache_scan_results_for_query(query)` | hpp 121 | true on cache hit (enter preload). |
| `void prepare_cache_for_scan_operators(scans)` | hpp 155 | create/verify per-operator cache entries. |
| `void set_scan_caching_enabled(level)` | hpp 128 | set/clear the cache level. |

## Types fundamental to *this* file

- **`cache_level`** *(Sirius; `op/scan/config.hpp`)* — `{NONE, PARQUET, TABLE_HOST,
  TABLE_GPU}`. **Think:** "how much of the scan to remember between runs."
- **`host_data_representation`** *(Sirius/cuCascade)* — fixed-width columnar data in host
  memory, built by the column builders; converted to a GPU `cudf::table` later. **Think:**
  "the CPU-side staged batch before it hits the GPU."
- **`duckdb_scan_task` + global/local state** *(Sirius)* — global state shares the DuckDB
  table-function state + `is_source_drained()`; local state holds the column builders.
  **Think:** "one scan task = one output batch, with shared progress across tasks."
- **`itask_executor`** *(Sirius parallel)* — the shared executor base (thread pool, queue,
  lifecycle); the GPU/scan/downgrade executors all fill in `manager_loop()`. **Think:**
  "the executor template; this is the host-I/O instantiation."
- **`memory_reservation_manager` / per-GPU `exclusive_stream_pool`** — host reservations +
  target-bound CUDA streams (the FIX-01 fix). **Think:** "reserve host RAM to read into;
  use a stream that belongs to the destination GPU."

## Takeaway

The scan executor is the source-side, host-I/O sibling of the GPU executor: same
reserve→pop→dispatch shape, plus a **four-level result cache** and **memory-proportional
multi-GPU targeting**. The actual reading lives in `duckdb_scan_task` (column builders →
`host_data_representation`, self-scheduling). Keep the rest of `scan.md` (scan manager,
`sirius::io`, Iceberg) as skim-on-demand. With scans, scheduling, and execution mapped,
the one remaining hard operator is the join —
[`../sirius_physical_hash_join.md`](../sirius_physical_hash_join.md).
