# Post-Pull Review — study notes vs. `sirius@d9172de6`

Date: 2026-06-15. Range reviewed: **`e65ca291..d9172de6`** (53 commits; ~90 files under the
dirs our maps cover). This report lists what changed *relative to the study maps* and what I
edited. Scope = our mapped files only; unrelated changes (perf, S3, telemetry internals) are
noted only where they shift cited lines.

## TL;DR

- **2 content corrections** (a map claim is now wrong): hash-join **MARK joins now use
  BUILD_PROBE**; the **`sirius::expression` PIMPL is retired** — the expression executor is
  now AST-native (the "#699 migration" we described is *done*).
- **1 type reclassification**: `sirius::join_condition` is **no longer a PIMPL over
  `duckdb::JoinCondition`** — it holds a Sirius AST node now.
- **Line drift** on most maps (small to moderate); the biggest are `sirius_context.cpp`
  (rewritten, ~+185-line shift on cited symbols) and `sirius_physical_hash_join.cpp` (~+32).
- **Our three flagged drift notes all still hold** (verified against code): pull-signal
  scheduler, `MAX_OOM_RETRIES = 100` (doc still says 10), and the
  `OnFinalizePrepare`/`PrepareInternal` chain.

## A. Content corrections (claims that are now wrong)

### A1. Hash join: MARK joins are now BUILD_PROBE-eligible — `[#924, #927]`
- **Was (our map):** "MARK joins have their own handling (and gate BUILD_PROBE off, cpp 391)."
- **Now:** `update_join_exec_mode` (cpp **411**) makes MARK joins **eligible for BUILD_PROBE**;
  `execute` builds a **persistent `cudf::filtered_join`** once on the right (filter) keys and
  probes it per batch (`#27`). New helpers `make_right_filtered_join` (68) /
  `make_right_filtered_join_ptr` (81). STANDARD MARK joins are now **adaptively built on the
  smaller side** (`#924`). New include `cudf/join/filtered_join.hpp`.
- **Edited:** `file-maps/op/sirius_physical_hash_join.md` (mode decision, Join-types section,
  cuDF-types list + filtered_join, line numbers).

### A2. Expression executor: PIMPL retired → AST-native — `[#701/#870, #703/#880, #922, #909, #928]`
- **Was (our map):** a "dual surface" — a DuckDB-typed path (`_expressions`,
  `duckdb::BoundXExpression` overloads, `sirius::expression` PIMPL) **and** a newer
  `sirius::ast::node` path, with an in-progress migration (issue #699).
- **Now:** the migration **landed**. `src/include/expression/expression.hpp` is **gone**;
  `gpu_expression_executor` is constructed from **`sirius::ast::node`** only — single member
  `_ast_expressions`, only `sirius::ast::*` per-alternative overloads (plus a **new
  `sirius::ast::aggregate` overload**). It still round-trips through `sirius::ast::to_duckdb`
  internally for result post-processing (return-type/expression-class recovery). The
  "tree of AST trees", three strategies, `min_ast_size`, and the `specializations/` fan-out
  (all 9 files still present) are unchanged.
- **Edited:** `file-maps/expression_executor/gpu_expression_executor.md` (replaced the
  "dual surface / #699" section with "migration complete"), `weeks/week3-concepts.md`
  (the "two parallel input surfaces" bullet).

### A3. `sirius::join_condition` is no longer a DuckDB PIMPL — `[#701, #703]`
- **Was:** "`sirius::join_condition` *(Sirius; PIMPL over `duckdb::JoinCondition`)*".
- **Now:** `expression/join_condition.hpp` forward-declares **`sirius::ast::node`** and holds
  it; `comparison_type` is a Sirius-native enum. It mirrors DuckDB's comparison semantics but
  is not a wrapper over `duckdb::JoinCondition`.
- **Edited:** the type entry in both `sirius_physical_hash_join.md` and
  `gpu_expression_translator.md`.

## B. Significant line drift (content intact, numbers updated where cited)

| Map | Symbol | Was → Now |
|---|---|---|
| `sirius_context.md` | `OnFinalizePrepare` (cpp) | 1044 → **859** |
| | `PrepareInternal` comment (cpp) | 1078 → **893** |
| | `registered_state->Insert` (cpp) | 1193 → **1008** |
| | `CanRequestRebind` / `OnFinalizePrepare` (hpp) | 109/113 → **98/102** |
| | `OnConnectionOpened` (hpp) | 454 → **343** |
| `sirius_physical_hash_join.md` | `update_join_exec_mode` | 382 → **411** |
| | `is_build_probe_mode` | 407 → **439** |
| | `get_next_task_hint` | 413 → **445** |
| | `get_next_task_input_data_for_build_probe` | 465 → **497** |
| | `execute` | 853 → **885** |
| `ungrouped_aggregate.md` | `build_aggregate_layout` | 66 → **114** |
| | `make_avg_column` | 214 → **231** |
| | `execute` | 322 → **267** |
| `transparent/sirius_optimizer_extension.md` | `OnFinalizePrepare` (cpp cross-ref) | 1044 → **859** |

`sirius_context.cpp` was substantially rewritten (≈355 lines; includes `rebind_stream`
support, `#918`) but the **mechanism we documented is intact** — capture-then-DuckDB-callback,
gated on `CanRequestRebind()`, registered under `"sirius_state"`.

### A4. (found during exhaustive line refresh) `task_scheduler` container types reverted
While refreshing every line citation, two `task_scheduler.hpp` member types had changed
back from our earlier read: **`_gpu_executors` is `std::unordered_map`** again (hpp 225,
was `std::map`) and **`_ready_devices` is `std::vector<int>`** (hpp 220, was `std::set`).
Note: the **in-source comment above `_gpu_executors` still says "std::map … deterministic"
— it is now stale** (contradicts the declared type). Under the pull-signal model this is
benign (ordering comes from `device_ready` arrival + binding device preference, not map
iteration). Corrected the "State worth knowing" bullets in `task_scheduler.md` and flagged
the stale comment. (Also new: `_task_queue_telemetry` member, `#809`.)

### Exhaustive line-citation refresh (follow-up pass)
Every line number in the line-drifted maps below was re-grepped and updated in place (not
just bannered): `task_scheduler.md`, `gpu_pipeline_executor.md`, `task_creator.md`,
`sirius_physical_ungrouped_aggregate.md`, `sirius_extension.md`, plus the remaining
citations in `sirius_physical_hash_join.md` and `sirius_context.md`. A full-text sweep for
old anchor values confirms none remain (the only matches are coincidences where a new line
equals a different old anchor).

### Other line-drifted maps — now fully refreshed (was: re-grep per caveat)
- `sirius_extension.md` — ~+3 lines (`GPUExecutionBind` 510→513, `GPUExecutionFunction`
  595→598, `RegisterGPUFunctions` 1175→1178); the 95-line change was S3/scan-manager cleanup
  (`#913/#923`). `run_internal_cpu_fallback_query` still the live fallback (100→**103**).
- `pipeline/task_scheduler.md` — `management_eventloop` 319→**276**; still the pull-signal
  matcher; RR-counter still vestigial. **Drift note still valid.**
- `pipeline/gpu_pipeline_executor.md` — `MAX_OOM_RETRIES = 100` at 273→**311**. **Doc-vs-code
  (10 vs 100) drift still valid.**
- `creator/task_creator.md` — heavily reorganized (503-line diff) by the scan-operator
  unification (`#871`) + Quent telemetry (`#809`), but the **hint-chain mechanism is intact**:
  `get_operator_for_next_task` 202→**135**, `manager_loop` 289→**203**, `schedule` 282→**196**.
- `transparent/sirius_optimizer_extension.md` — the optimizer-hook lines barely moved
  (`gpu_execution_enabled` 31, `sirius_pre_optimizer_hook` 40, `sirius_optimizer_hook` 74).

## C. Scan subsystem was restructured — `[#871, #895, #913, #936]`
The GPU scan paths were **unified behind a `gpu_ingestible` abstraction**: `parquet_scan_task.cpp`,
`sirius_gpu_parquet_scan_operator.cpp`, and `sirius_gpu_duckdb_native_scan_operator.cpp` were
**deleted**, replaced by `sirius_gpu_scan_operator.cpp` + `{parquet,duckdb_native,pinned_table}_gpu_ingestible.cpp`
and `row_group_metadata.hpp`.
- **Our `op/scan/duckdb_scan_executor.md` map's subject file did *not* change** (the host-I/O
  scan executor is intact). But its "four scan paths" scope-note and the surrounding
  scan-manager pointers are now partly stale. **Edited:** added a short "since this pull" note;
  did not rewrite (the executor itself is the mapped target and is unchanged).

## D. Ungrouped aggregate accuracy fix — `[#926]`
`make_avg_column` now keeps **decimal AVG out of `long double`** — it divides on-device rather
than round-tripping decimals through host `long double`, avoiding precision loss. Our map's
two-phase / AVG-decomposition description is unchanged; added a one-line accuracy note + line
updates.

## E. Verified unchanged (no edit needed)
- Memory: `sirius_memory_reservation_manager` still the thin cuCascade subclass (hpp +12 lines,
  cpp +6 — cosmetic); `defragmenter_oom_policy` intact.
- Downgrade: `downgrade_executor` request/predicate model intact (+102 cpp lines are mostly
  telemetry/`#809`); tiered candidate fetch unchanged.
- Operator base: `TaskCreationHint {WAITING_FOR_INPUT_DATA, READY}`, `task_creation_hint`,
  `get_next_task_hint`/`get_next_task_input_data` all present.
- Fallback: live path (`run_internal_cpu_fallback_query`) + per-operator `NotImplementedException`
  + S3-no-fallback all intact; legacy `src/legacy/fallback.cpp` still the dead one.

## Edits applied (files touched)
- `file-maps/op/sirius_physical_hash_join.md` (A1, A3, B)
- `file-maps/expression_executor/gpu_expression_executor.md` (A2)
- `file-maps/expression_executor/gpu_expression_translator.md` (A3)
- `weeks/week3-concepts.md` (A2)
- `file-maps/sirius_context.md` (B)
- `file-maps/transparent/sirius_optimizer_extension.md` (B)
- `file-maps/op/sirius_physical_ungrouped_aggregate.md` (B, D)
- `file-maps/op/scan/duckdb_scan_executor.md` (C)
- `_claude-methodology.md` (review log)
