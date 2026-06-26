# `op/scan/sirius_gpu_scan_operator.cpp` → GPU-Scan-Operator Map

The **unified GPU scan source operator** introduced by `#871`. It replaced the per-format
`sirius_gpu_parquet_scan_operator` and `sirius_gpu_duckdb_native_scan_operator` with one
format-agnostic operator that carries **no decode code of its own**: it pulls splits from a
`split_connector` and delegates all real work (`materialize_table`, `post_filter_and_project`)
to an installed `io::gpu_ingestible`. This is the **consumer half** of the GPU-native scan — the
counterpart to the producer-half [`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md).

This map matters because it is the live home of the scan's `execute()` and the `filter_state`
skip decision that the FILTER→PROJECTION gather ticket turns on (see the walkthrough
[`../../../notes/walkthroughs/scan-pipeline-and-filter-project.md`](../../../notes/walkthroughs/scan-pipeline-and-filter-project.md)
§6-7 and the concept note [`../../../notes/expressions/fused-kernels-and-breakers.md`](../../../notes/expressions/fused-kernels-and-breakers.md) §5).

> Paths relative to the Sirius repo root. Lines as of 2026-06-26 — re-grep if the file moved.
> Post-`#871`: `SiriusPhysicalOperatorType::GPU_SCAN`. The older per-format operators and the
> bypassed `sirius_physical_table_scan::execute()` are stale — see that map for the dead copy.

## Where this sits

```
scan_manager::prepare_for_query                     [producer half]
   ├ op->take_table_info()        ──► builds ingestible
   ├ op->install_ingestible(…)     ──► _ingestible
   ├ op->set_split_connector(new)  ──► _split_connector (fresh, open)
   └ split_provider drives the SAME ingestible → pushes splits → connector
                                  │
   sirius_gpu_scan_operator       ▼   [consumer half — THIS file]
     get_next_task_hint()  → READY while connector open          :108
     get_next_task_input_data() → ingestible->consume_next_input :125
     execute(split)                                              :165
        materialize_table(md.scan())                             :187
        if filter && state != ROW_FILTERED_AND_PROJECTED:
            post_filter_and_project(...)                         :189
        make_data_batch → pipelineable_operator_data             :227
```

The operator is the pipeline **source**. The scan_manager installs both the ingestible and the
connector; the operator's job at runtime is purely: pull a split → drive the ingestible → wrap
the resulting `cudf::table` as a `data_batch` for the downstream pipeline.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor | :87-101 | Stores `_table_info`, then **starts with a *closed* connector** (:100) so an operator wired into a pipeline but never prepared (e.g. CPU fallback) reports `all_ports_empty()` and yields no task hint. scan_manager swaps in a fresh open connector during prepare. |
| `get_next_task_hint` | :108-116 | Returns `READY` whenever the connector is open — **even if the queue is empty**; the dispatched worker parks in `split_connector::get_next_split` until a split arrives or the connector closes. `nullopt` once closed. |
| `all_ports_empty` | :118-123 | Done = connector closed **and** (no ingestible **or** `ingestible->consumer_drained()`). The unprepared path has no ingestible. |
| `get_next_task_input_data` | :125-132 | Pulls the next input: delegates to `ingestible->consume_next_input(connector)` (coalescing); the unprepared path pulls a raw split straight from the connector. |
| `take_table_info` / `peek_table_info` | :137 / hpp :136 | scan_manager handshake: **peek** to match pinned-cache entries, then **take** to build the ingestible. After `take`, peek dereferences null — peek-before-take. |
| `install_ingestible` / `get_ingestible` | :142 / :147 | Install the one shared ingestible (drives both this operator's `execute` and the provider's split emission). `get_ingestible` throws if prepare hasn't run. |
| `set_split_connector` / `get_split_connector` | :157 / hpp :152 | The connector swap point; scan_manager installs a fresh open one. |
| **`execute`** | :165-231 | The per-split body. Two input shapes (below). For a **fresh** read: `materialize_table` then — only if `has_filter()` **and** the reader didn't already do it (`state != ROW_FILTERED_AND_PROJECTED`) — `post_filter_and_project` (:188-190). For a **pinned** hit: forward unchanged when no `filter_info` (zero-copy, :202-206), else `post_filter_and_project` on the released cached table (:218). Wraps the result in a `data_batch` (:227). |
| `no_history_peak_memory_estimate` | :233-243 | Cold reservation heuristic: resident (pinned) inputs ≈ input size; fresh reads ×8 (decompress+decode expansion upper bound). |

### The `prepare_for_processing` half (`scan_operator_with_pinned_table_input`, :46-82)

| Function | Lines | Role |
|---|---|---|
| `prepare_for_processing` | :46-70 | For a **pinned** input: if the cached batch is host-tier, convert it onto the requested GPU memory space via the converter registry (GPU-tier batches need no work); then capture the resident `memory_space` so `execute` tags the output correctly. |
| `get_estimated_size_in_bytes` | :72-82 | Generic representation accessor — works for both GPU-resident and host batches that will be upgraded. |

## The two input shapes (`sirius_gpu_scan_operator_data.hpp`)

The connector only ever carries these two `operator_data` types; `execute` `dynamic_cast`s on them:

- **`scan_operator_input`** — a **fresh read** split. Carries `io::scan_and_filter_metadata`
  (`scan()` descriptor + optional `filter_and_project()`); `prepare_for_processing` just records
  the target `gpu_memory_space`. Drives `materialize_table` (+ conditional post-filter).
- **`scan_operator_with_pinned_table_input`** — a **pinned-cache hit**. Carries a zero-copy
  `data_batch` over pinned columns + optional `post_filter_and_projection_info`. Null filter →
  forwarded unchanged; non-null → `post_filter_and_project` on the cached view.

## Types fundamental to *this* file

- **`io::ingestible_table_info`** *(Sirius io)* — per-table bind data parked at construction;
  consumed by scan_manager (peek to match pins, take to build the ingestible).
- **`io::gpu_ingestible`** *(Sirius io)* — the installed source that does the real work
  (`materialize_table`, `post_filter_and_project`, `consume_next_input`, `consumer_drained`). See
  [`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md).
- **`scan_manager::split_connector`** *(Sirius)* — the producer→consumer queue; closed-state
  drives the source's done/hint logic.
- **`io::filter_state`** *(Sirius io)* — `ROW_FILTERED_AND_PROJECTED` is the value that makes
  `execute` **skip** `post_filter_and_project` (the reader already filtered+projected). This is
  the ticket-relevant branch.
- **`scan_operator_input` / `scan_operator_with_pinned_table_input`** *(Sirius, sibling header)*
  — the two split shapes above.
- **`data_batch` / `gpu_table_representation`** *(cucascade)* — output wrapper + the GPU table
  the pinned path releases.

## How it fits with the rest of the scan stack

- **Producer half** ([`../../scan_manager/sirius_scan_manager.md`](../../scan_manager/sirius_scan_manager.md)):
  builds the ingestible, installs it + a fresh connector, fire-and-forgets the split_provider that
  feeds this operator's connector. The two meet at `_split_connector`.
- **The ingestible** ([`duckdb_native_gpu_ingestible.md`](duckdb_native_gpu_ingestible.md)): where
  `materialize_table` / `post_filter_and_project` actually run — the gather the ticket edits.
- **Planner** ([`../../planner/sirius_plan_get.md`](../../planner/sirius_plan_get.md)): the
  keep-set (`projection_ids` → `output_arity`) the ingestible applies originates here.
- **Bypassed sibling** ([`../sirius_physical_table_scan.md`](../sirius_physical_table_scan.md)):
  the pre-`#871` `execute()` copy this operator superseded — read it only to recognize it's dead.
- **Other scan family** ([`duckdb_scan_task.md`](duckdb_scan_task.md)): the CPU-staging
  `DUCKDB_SCAN` path — orthogonal; don't conflate `GPU_SCAN` (here) with `DUCKDB_SCAN` (there).

## Takeaway

`sirius_gpu_scan_operator` is the format-agnostic GPU scan **source**: it owns no decode logic,
just the scheduling surface (hint/drain/pull) and a thin `execute` that routes each split through
the installed ingestible and wraps the result. The one decision it makes that the ticket cares
about is the `filter_state` skip (:188): run `post_filter_and_project` only when the reader didn't
already filter+project. Read with the scan_manager map (who feeds it) and the ingestible map
(what it delegates to).
