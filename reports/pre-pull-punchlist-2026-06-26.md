# Pre-Pull Punch-List — study maps vs. `sirius@d1960b13` (origin/dev)

Date: 2026-06-26. Range: **`d9172de6..d1960b13`** (38 commits; Jun 15 → Jun 26).
Status: **NOT yet pulled** — local tree is still at `d9172de6`. This is a forward-looking
punch-list to apply *after* pulling, so the maps stay accurate for the current HEAD until then.
Scope = mapped files only; the large GPU-kernel / S3 churn is noted but unmapped (skim-tier).
Companion to [`post-pull-review-2026-06-15.md`](post-pull-review-2026-06-15.md).

## TL;DR

- **No architectural change.** The scan flow (`GPU_SCAN` + `gpu_ingestible` + `scan_manager`
  producer/consumer) and the planner/converter (`split_table_scan_source`,
  `construct_sirius_specific_operator`) are **untouched** — converter and `sirius_plan_get`
  show **zero** diff. All the vestigial-`DUCKDB_SCAN` corrections from this session stay correct.
- **3 substantive map edits**, all one theme: **keeping a partitioned operator's tasks pinned to
  one real GPU** across creation *and* OOM reschedule (`task_creator`, `gpu_pipeline_executor`),
  plus a defensive null-guard in `gpu_expression_executor`.
- **Everything else is cosmetic**: the `cucascade/data/*` → `cucascade/cudf/*` include rename
  (the cudf-free-core split, `#989`) across ~12 mapped files, + one cudf version guard in
  hash_join (`26.8`→`26.6`). Re-grep absorbs the ≤ +1-line drift.
- **Build caveat:** `#989` is a cucascade dependency bump ("build against PR #150 cudf-free-core
  split"). Budget `pixi run make clean && make`, not a docs rewrite.

## How to apply (after pulling)

1. `git pull` (or `git merge origin/dev`) on `dev`.
2. `pixi run make clean && pixi run make` — the `#989` dep bump needs a clean rebuild.
3. Apply the 3 edits in §A below.
4. Re-grep the cited line numbers in the §B files (mechanical; ≤ +1 line each).
5. Optional: fold the §A device-pinning theme into the multi-GPU sections.

## A. Substantive edits (a map claim gains/needs new behavior)

### A1. `task_creator` — partition→GPU pin now indexes the *active executor set* — `[device-pinning]`
- **New:** member `_active_gpu_ids` (sorted, deduped device ids that actually have a GPU
  executor — from `mem_res_mgr.get_memory_spaces_for_tier(GPU)`), populated in the ctor.
- **Change:** in `manager_loop`, the partition device-pin for `partitioned_operator_data` now
  indexes `_active_gpu_ids[partition_idx % _active_gpu_ids.size()]` instead of
  `_sys_topology->gpus[...]`. Fixes phantom pins when `num_gpus < physical GPU count` (a
  partition task could otherwise pin to a device with no executor).
- **Apply:** `file-maps/creator/task_creator.md` — multi-GPU placement description + the types
  list (`_active_gpu_ids`). Line refs: member in `task_creator.hpp` (~:210), ctor populate
  (~:64), `manager_loop` use (~:290). *(Re-grep after pull.)*

### A2. `gpu_pipeline_executor` — two OOM-path fixes — `[device-pinning, OOM]`
- **Change 1 (reservation clamp):** in `manager_loop`, `bytes_needs` is now clamped to
  `_memory_space->get_max_memory()` before `make_reservation()`. Without it, a runaway
  history-based peak estimate (small input that once drove a near-cap peak → huge
  peak/estimate ratio) could exceed capacity, yield only a partial reservation, and **livelock
  the OOM-reschedule loop** until the retry cap failed the query. Telemetry still reports the
  pre-clamp estimate.
- **Change 2 (pin preservation):** on OOM reschedule, the new `local_state` now copies
  `get_preferred_device_id()` from the current local state. Dropping it let an OOM'd partition
  task **scatter to the wrong GPU and touch a cuco table built on another device**
  (`cudaErrorInvalidValue`).
- **Apply:** `file-maps/pipeline/gpu_pipeline_executor.md` — OOM-retry section gains both. Line
  refs: clamp (~:137–160 of `manager_loop`), pin copy (~:372–380, reschedule branch).
  *(Re-grep after pull.)*

### A3. `gpu_expression_executor` — null-AST guard in `execute()` — `[defensive]`
- **Change:** `execute(table_view)` now `throw`s a `duckdb::InternalException` if an
  `_ast_expressions` slot is null (means `from_duckdb` declined an unsupported expression but the
  planner still routed it to the GPU). Hard-fails instead of segfaulting; flagged as a planner
  bug to fix (no CPU fallback).
- **Apply:** `file-maps/expression_executor/gpu_expression_executor.md` — small note in the
  `execute()` description. Line ref: top of the `_ast_expressions` loop (~:312). *(Re-grep.)*

## B. Cosmetic — re-grep only, no content change

Pure `cucascade/data/{cpu,gpu}_data_representation.hpp` → `cucascade/cudf/{host,gpu}_data_representation.hpp`
include rename (≤ +1 line drift, near the include block), plus in scan files a
`column_metadata.type_id = cudf::type_id::X` → `static_cast<int32_t>(...)` (the field became a
plain `int32_t`). Mapped files affected:

- `sirius_extension`, `sirius_context`, `sirius_physical_limit`, `sirius_physical_top_n`,
  `sirius_physical_ungrouped_aggregate` — include rename only.
- `sirius_physical_hash_join` — **only** a cudf version guard `26.8`→`26.6` (no logic change).
- Scan files (already current per this session's corrections): `sirius_gpu_scan_operator`,
  `pinned_table_gpu_ingestible`, `duckdb_scan_task`, `duckdb_scan_executor`,
  `sirius_scan_manager`, `sirius_physical_table_scan`.

## C. Big churn to ignore (unmapped, skim-tier)

- **`src/cuda/scan/` GPU string-decode refactor** (~2,000 lines): FSST + dictionary decoders
  split into new `src/cuda/scan/strings/` (`fsst.cu`, `dict_fsst.cu`, `dictionary.cu`,
  `uncompressed.cu`, + `detail/*.cuh`). These are the kernels `materialize_table` / native-decode
  dispatch *into* — **the control flow the scan maps describe is unchanged**, only the kernel
  internals below it moved. No map covers these at current depth.
- **`src/io/s3/`** (`s3_reactor.cpp` +107, `s3_ioctx.cpp` +85) — S3 backend internals, skim-tier.
- `src/expression/function_id.cpp` (+33), `src/sirius_ffi.cpp` (+13), telemetry — unmapped.

## D. Re-verify after pull (the claims this session established)

Re-run to confirm the vestigial-`DUCKDB_SCAN` findings still hold (none of these files changed in
this range, so they *should* be byte-identical — confirm anyway):

```
# DUCKDB_SCAN built only by the (dead) construct_sirius_specific_operator
grep -rn "construct_sirius_specific_operator" src | grep -v src/legacy
# split_table_scan_source still routes seq_scan → GPU_SCAN
grep -n "seq_scan\|insert_duckdb_native_scan_operator\|GPU_SCAN" src/pipeline/sirius_pipeline_converter.cpp
# task_scheduler still builds NO duckdb_scan_executor
grep -rn "duckdb_scan_executor\|scan_cfg" src/pipeline/task_scheduler.cpp
# task_creator still has no DUCKDB_SCAN/duckdb_scan_task branch
grep -rn "DUCKDB_SCAN\|duckdb_scan_task" src/creator/task_creator.cpp
```
