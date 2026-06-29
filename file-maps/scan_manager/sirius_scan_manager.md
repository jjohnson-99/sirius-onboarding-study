# `scan_manager/sirius_scan_manager.cpp` → Scan-Manager Map

The **split-production engine** that feeds the unified GPU scan operators. It is a *separate*
subsystem from the operator/pipeline path: per query it walks the plan's `GPU_SCAN` sources,
builds a `gpu_ingestible` for each, installs it on the operator, and then fans metadata workers
out over its own thread pool (fire-and-forget) to push **splits** into each operator's
`split_connector`. The operator side then pulls those splits and runs decode + filter/project.

This map exists because the scan walkthrough
([`../../notes/walkthroughs/scan-pipeline-and-filter-project.md`](../../notes/walkthroughs/scan-pipeline-and-filter-project.md)
§4) leans on this file for "where splits come from," and because it owns the IO backends the
whole scan stack reads through.

> Paths relative to the Sirius repo root. Lines as of 2026-06-26 — re-grep if the file moved.
> Post-`#871` (scan unification → `GPU_SCAN` + `gpu_ingestible`) and post-`#913` (the
> scan_manager now **owns** the IO backends / S3 pool / prefetch buffer pool). Earlier docs
> that show per-format scan operators or externally-owned IO contexts are stale.

## Where this sits

```
sirius_engine (per query)
   └─ scan_manager.prepare_for_query(query, gpu_memory_spaces)   :222
        walk pipelines → for each GPU_SCAN source:
          op->take_table_info()                                   :259
          _factory.produce(table_info, …)  → gpu_ingestible       :262
          op->install_ingestible(…)                               :265
          provider = split_provider(op->get_ingestible())         :270
          op->set_split_connector(new split_connector)            :277
        start_metadata_processing()                               :287
           └─ provider->run(dispatcher, connector)  [fire-and-forget]  :302
                 … worker threads produce splits → connector …
                                  │
                                  ▼  (operator side, different subsystem)
   sirius_gpu_scan_operator::execute()  pulls from its split_connector,
   then materialize_table → post_filter_and_project
```

The scan_manager is constructed once (owned by `SiriusContext`) and reused across queries;
`prepare_for_query` / `reset` is the per-query cycle. It does **not** decode data itself — it
produces *split descriptors* (row-group ranges + scan/filter metadata) and hands the actual
read+decode to the ingestible the operator drives.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor | :76-143 | Builds and **owns** the IO backends. Local `uring_ioctx` is constructed **unconditionally** (:90) — duckdb-native GPU scan needs it for host reads even when `use_sirius_datasource=false`. Optional S3 backend (async `s3_ioctx` or blocking `s3_blocking_ioctx`, :97-127). Optional prefetch `buffer_pool` + `initialize_cache` on each backend (:129-140). |
| `~sirius_scan_manager` | :145-152 | `stop()` then tears down owned resources in order: `_io_ctxs.clear()` → S3 pool stop → prefetch pool reset (the `#913` ownership). |
| `describe_parquet` | :182-220 | Bind-path schema probe. Footer-only fetch + Thrift parse via `hybrid_scan_reader`; inserts a **metadata-only** cache entry (empty ranges) so the later scan's `get_metadata` hits and the footer is parsed once. Entry point behind `sirius_read_parquet`'s bind. |
| **`prepare_for_query`** | :222-288 | Per-query setup. `reset()`, refresh caches, build one shared `round_robin_strategy` over the sorted GPU set (:242-248), then walk pipelines: skip non-`GPU_SCAN` sources (:254), `take_table_info` (:259), `_factory.produce` the ingestible (:262), `install_ingestible` (:265), build `split_provider` borrowing the ingestible (:270), stamp the round-robin strategy (:276), give the op a fresh `split_connector` (:277), register (:278-279). Ends by calling `start_metadata_processing` (:287). |
| `start_metadata_processing` | :290-311 | For each registered op, `provider->run(*_dispatcher, *connector)` — **fire-and-forget** (:302): enqueues workers and returns. Worker exceptions ride `connector.close(exception_ptr)` and surface when the consumer drains. A synchronous throw from `run()` is forwarded via `connector->close(current_exception())` (:308). |
| `reset` | :313-321 | `request_stop` + `wait_for_all` on the dispatcher, clear the op/provider maps, rebuild a fresh dispatcher. The per-query teardown half of the cycle. |
| `start` / `stop` | :323-334 | `start` is a no-op; `stop` does `reset()` then halts the orchestration thread pool (backends stay alive to serve in-flight reads — torn down only in the dtor). |
| `create_datasource` | :493-512 | Resolve a path to the first owned backend whose `supports(path)` is true; build a `sirius_datasource`. `use_sirius_datasource=false` deliberately leaves **local** paths unclaimed (→ KvikIO fallback); object-store paths always resolve. |
| `insert_pinned_entry` / `_host` | :336-491 | Populate `_pinned_entries` for `CALL pin_table`. GPU tier: per-column chunk vectors with per-chunk `memory_space` alignment + same-row-count merge semantics. HOST tier: one `host_data_representation` per batch. Feeds `pinned_table_gpu_ingestible`. |
| `remove_pinned_entry` | :514-517 | Drop a pin. |

## Types fundamental to *this* file

- **`scan_manager_config`** *(Sirius, hpp)* — thread-pool sizing, `use_sirius_datasource`,
  uring knobs, prefetch-cache knobs (`enable_prefetch_cache`, `prefetch_buffer_pool_bytes`),
  `enable_chunk_prewarm`, and optional `s3_config` / `s3_use_async_backend`.
- **`pinned_entry`** *(Sirius, hpp)* — one pinned table: `column_names`, resolved `file_paths`,
  GPU `data_batches_by_column` + parallel `chunk_memory_spaces`, or HOST `host_chunks`; `tier`,
  `num_rows`, `is_partial`. Consumed by `pinned_table_gpu_ingestible`.
- **`parquet_bind_result`** *(Sirius, hpp)* — `describe_parquet` output: `return_types`,
  `names`, `object_size`, `total_num_rows` for the table function's bind out-params.
- **`split_provider`** *(Sirius)* — the per-op worker driver this starts; produces splits onto
  the connector. See [`split_provider`](split_provider.md).
- **`split_connector`** *(Sirius)* — the hand-off queue between provider workers and the
  operator's consumer; `close(exception_ptr)` carries both EOF and errors. See
  [`split_connector`](split_connector.md).
- **`gpu_ingestible` / `gpu_ingestible_factory`** *(Sirius)* — the per-source decode+filter
  object and its builder. The factory borrows `_pinned_entries` (declared last so its ctor
  reference binds to a fully-constructed member). See
  [`../op/scan/duckdb_native_gpu_ingestible.md`](../op/scan/duckdb_native_gpu_ingestible.md).
- **`sirius_ioctx` / `uring_ioctx` / `s3_ioctx`** *(Sirius io)* — the owned IO backends in
  `_io_ctxs`, priority-ordered (local first, then object-store).
- **`round_robin_strategy`** *(Sirius)* — spreads fresh-read splits across the query's GPUs;
  one instance shared across all providers so placement is even over the whole scan stage.

## How it fits with the rest of the scan stack

- **Operator side** ([`../op/scan/duckdb_native_gpu_ingestible.md`](../op/scan/duckdb_native_gpu_ingestible.md),
  and `sirius_gpu_scan_operator`): the scan_manager produces the splits; the operator consumes
  them and runs decode + `post_filter_and_project`. The two meet at the `split_connector`.
- **Planner side** ([`../planner/sirius_plan_get.md`](../planner/sirius_plan_get.md)): the
  `table_info` the scan_manager hands the factory carries the planner's keep-set
  (`projection_ids` → `output_arity`).
- **Walkthrough** ([`../../notes/walkthroughs/scan-pipeline-and-filter-project.md`](../../notes/walkthroughs/scan-pipeline-and-filter-project.md)):
  §4 (split production) is exactly `prepare_for_query` + `start_metadata_processing`.

## Takeaway

The scan_manager is the **producer half** of the scan: a query-scoped orchestrator that owns the
IO backends, builds one ingestible per `GPU_SCAN` source, and fire-and-forgets metadata workers
that stream splits into per-operator connectors. It decides *what to read and where* (round-robin
GPU placement, pinned-cache reuse) but never decodes or filters — that is the operator/ingestible
half. Read alongside `split_provider` / `split_connector` (the worker + queue) and the ingestible
maps (the consumer).
