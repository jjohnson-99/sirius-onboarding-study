# `op/scan/pinned_table_gpu_ingestible.cpp` → Scan Map

The **pinned-cache** `gpu_ingestible` implementation (`#871`). It serves a scan from an
already-resident `CALL pin_table` entry instead of reading from storage. **Still live** — it is
built by [`gpu_ingestible_factory::try_cached`](../../scan_manager/sirius_scan_manager.md) when a
pinned entry's file paths match the scan, short-circuiting the per-format
[`parquet_gpu_ingestible`](parquet_gpu_ingestible.md). ~226 lines.

> Paths relative to the Sirius repo root. Lines as of 2026-06-26 — re-grep if the file moved.

## Where this sits

```
gpu_ingestible_factory::try_cached  (pinned entry matches file paths)
   ├ GPU-tier ctor (columns_per_request + chunk_memory_spaces)   :50
   └ HOST-tier ctor (host_chunks + column_indices + gpu spaces)   :81
            │  installed on sirius_gpu_scan_operator
            ▼
   next_split_provider → produce_batch → scan_operator_with_pinned_table_input  :119 / :148
            │  (carries a zero-copy data_batch + optional filter_info)
            ▼
   sirius_gpu_scan_operator::execute  (pinned branch)
      no filter_info → forward batch unchanged (zero-copy)
      filter_info    → post_filter_and_project (filter gather + assembly)  :194
```

Unlike the fresh-read ingestibles, the cached path **never calls `materialize_table`** — that
override throws (:181). The split it emits is a `scan_operator_with_pinned_table_input`, which the
operator routes straight into `post_filter_and_project`.

## Two tiers, two constructors

| Ctor | Lines | Input |
|---|---|---|
| **GPU-tier** | :50-79 | `columns_per_request` (per-D-column chunk vectors, already GPU-resident) + parallel `chunk_memory_spaces`. Transposes columns→batches; validates equal chunk counts. |
| **HOST-tier** | :81-110 | `host_chunks` (one `host_data_representation` per batch) + `column_indices` (slice map) + `gpu_memory_spaces` (upload targets). Requires a non-empty GPU-space map. |

The factory picks the tier from `pinned_entry::tier`; both store their chunks into a
`std::variant<gpu_chunk, host_chunk> _batches` and share the rest of the flow.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| GPU-tier ctor | :50-79 | Transpose `columns_per_request` (per-column → per-batch `gpu_chunk`); validate uniform chunk counts. |
| HOST-tier ctor | :81-110 | Store host chunks; require non-empty `gpu_memory_spaces` (needed so `produce_batch` can upload). |
| `has_more_splits` / `next_split_provider` | :114 / :119 | Atomically claim the next batch; precompute `apply_filter` (filter present) and `apply_assembly` (`needs_output_assembly`); return a callable producing one `scan_operator_with_pinned_table_input` carrying the batch + (optional) `pinned_table_post_filter_and_projection_info`. |
| `produce_batch` | :148-179 | `std::visit` on the variant: **GPU** → wrap the columns as a `gpu_table_representation` on the chunk's memory space (zero-copy, co-owned via `shared_ptr<column>`); **HOST** → slice the host chunk by `_column_indices` and emit a host-resident batch (upload deferred to `prepare_for_processing`). |
| `materialize_table` | :181-192 | **Unreachable** — throws. The cached path uses the pinned-input + `post_filter_and_project` route only. |
| `post_filter_and_project` | :194-224 | The cached post-processing: if `apply_filter`, run `gpu_expression_executor::select` (the **gather**, :207); if `apply_assembly`, `assemble_scan_output` to the plan layout (:217, partition branch never hit — cached path is gated against hive partitions upstream). |

## Types fundamental to *this* file

- **`pinned_table_post_filter_and_projection_info : io::post_filter_and_projection_info`** *(hpp)*
  — carries `apply_filter` / `apply_assembly` flags computed at split emission.
- **`chunk_variant = variant<gpu_chunk, host_chunk>`** *(hpp)* — `gpu_chunk` =
  `vector<shared_ptr<cudf::column>>`; `host_chunk` = `shared_ptr<host_data_representation>`.
- **`scan_plan`** *(Sirius)* — shared read/output layout, used by `produce_batch` (column order)
  and `post_filter_and_project` (assembly). See [`scan_plan.md`](scan_plan.md).
- **`gpu_expression_executor`** *(Sirius)* — runs the filter `select` (gather) on the cached
  batch. Same engine as FILTER/PROJECTION.
- **`pinned_entry`** *(Sirius scan_manager)* — the source of the cached chunks + memory spaces;
  built by `insert_pinned_entry` / `_host`. See [`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md).
- **`gpu_table_representation` / `host_data_representation` / `data_batch`** *(cucascade)* — the
  resident wrappers `produce_batch` emits; the host one is upgraded on GPU later.
- **`scan_operator_with_pinned_table_input`** *(Sirius)* — the split shape this emits (vs
  `scan_operator_input` for fresh reads). See [`sirius_gpu_scan_operator.md`](sirius_gpu_scan_operator.md).

## How it fits with the rest of the scan stack

- **Factory** ([`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md)):
  `gpu_ingestible_factory::try_cached` decides whether a pinned entry matches and which tier ctor
  to call (with partial-pin / hive-partition / missing-column fall-throughs to the per-format path).
- **Operator** ([`sirius_gpu_scan_operator.md`](sirius_gpu_scan_operator.md)): the pinned branch
  of `execute` — forwards unchanged when `filter_info` is null, else calls
  `post_filter_and_project`; `prepare_for_processing` does the host→GPU upload.
- **Per-format sibling** ([`parquet_gpu_ingestible.md`](parquet_gpu_ingestible.md)): what this
  replaces on a cache hit (and what the factory falls through to on a miss).

## Takeaway

`pinned_table_gpu_ingestible` is the cache-hit source: it hands out already-resident chunks
(GPU-tier zero-copy, or HOST-tier sliced + uploaded later) as `scan_operator_with_pinned_table_input`
splits, skips `materialize_table` entirely, and only runs filter+assembly when needed in
`post_filter_and_project`. Its gather (when a filter is present) is the `select` at :207 — the
cached analogue of the FILTER→PROJECTION ticket. Read with the factory (who builds it) and
[`scan_plan.md`](scan_plan.md) (the layout it serves).
