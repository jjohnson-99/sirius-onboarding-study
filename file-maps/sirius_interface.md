# `sirius_interface.cpp` → Execution-Flow Map

Companion for Week 2, Days 1–2 of [`onboarding-path.md`](../onboarding-path.md): read
`src/sirius_interface.cpp` (and its header `src/include/sirius_interface.hpp`) with
`docs/super-sirius/execution-flow.md` open. Tagged *read* in the plan, so this map
goes a level deeper than the `sirius_extension` one — it walks the call graph and the
state machine, not just the function-to-step mapping.

> Source/doc paths (`src/...`, `docs/...`) are relative to the Sirius repo root — the
> `sirius/` checkout beside the `sirius-onboarding-study/` folder. Line numbers were
> accurate as of the read on 2026-06-10; re-grep the function name if the file moved.

## The key idea first

**This file is a hand-rolled fork of DuckDB's own query-execution methods.** Nearly
every function carries a comment like *"based on `ClientContext::Query`"* or
*"based on `PendingQueryResult::ExecuteInternal`."* DuckDB normally drives a query
through `ClientContext` → pending statement → execute → fetch; Sirius copied that
skeleton into `sirius_interface` and **swapped DuckDB's CPU executor for the Sirius
GPU engine**. So if you know (or skim) how DuckDB's `ClientContext::Query` works, you
already know the shape of this file — the only real change is *who executes*.

It implements **Step 3 (Query Lifecycle Setup)** and **Step 9 (Result Extraction)**
of the execution-flow doc. It is the layer between the entry doorways
(`sirius_extension.cpp` / `physical_sirius_execution.cpp`) and the engine
(`sirius_engine.cpp`).

## The two-phase model (why there are so many functions)

DuckDB splits running a query into **prepare-a-pending-result** then **execute-it**,
and Sirius mirrors that:

1. **Pending phase** — bind parameters, build the engine + pipelines, hand back a
   `PendingQueryResult`. *Nothing has run on the GPU yet.* Setup/plan errors surface
   here, cleanly separated from execution errors.
2. **Execute phase** — actually run the pipelines, then fetch the materialized result.

That separation is why one logical "run the query" spreads across
`sirius_execute_query` → `…_pending_statement_or_prepared_statement` →
`…_pending_statement_internal`, then `…_execute_pending_query_result` →
`fetch_result_internal`. It looks like a lot of indirection; it's really just DuckDB's
two-phase contract reproduced.

## Call graph (top to bottom)

`sirius_execute_query` is the entry point called from Step 1b
(`GPUExecutionFunction`) and from the transparent path (`PhysicalSiriusExecution`).

| Function (signature) | Line | Role / doc step |
|---|---|---|
| `unique_ptr<QueryResult> sirius_execute_query(ClientContext&, const string& query, shared_ptr<sirius_prepared_statement_data>&, const PendingQueryParameters&)` | 214 | **Entry / Step 3.** *Based on `ClientContext::Query`.* Wraps the whole flow in try/catch so any throw still runs `cleanup_internal`. Calls the pending builder, then the executor. |
| `unique_ptr<PendingQueryResult> sirius_pending_statement_or_prepared_statement(ClientContext&, const string& query, shared_ptr<sirius_prepared_statement_data>&, const PendingQueryParameters&)` | 131 | **Step 3 (pending phase).** Calls `begin_query_internal`, then `sirius_pending_statement_internal`; returns early on error. |
| `void begin_query_internal(const string& query)` | 83 | Opens the per-query state: constructs `sirius_active_query` (the state struct — see below). Asserts no query is already active (one-at-a-time invariant). |
| `unique_ptr<PendingQueryResult> sirius_pending_statement_internal(ClientContext&, shared_ptr<sirius_prepared_statement_data>&, const PendingQueryParameters&)` | 150 | **The heart of Step 3.** *Based on `ClientContext::PendingPreparedStatementInternal`.* Binds params, **creates the `sirius_engine`** (160), **creates the `sirius_physical_materialized_collector`** result sink (166), calls **`engine.initialize(collector)`** (177) to build pipelines (→ Step 4, `sirius_engine.cpp`), and returns a `PendingQueryResult`. |
| `unique_ptr<QueryResult> sirius_execute_pending_query_result(PendingQueryResult&)` | 189 | **Step 5 trigger.** *Based on `PendingQueryResult::ExecuteInternal`.* Validates, then calls **`engine.execute()`** (197) — the blocking GPU run — and on success calls `fetch_result_internal`. |
| `unique_ptr<QueryResult> fetch_result_internal(PendingQueryResult&)` | 91 | **Step 9.** Calls **`engine.get_result()`** (102) to pull the materialized result, then `cleanup_internal`. |
| `void cleanup_internal(BaseQueryResult*, bool invalidate_transaction)` | 107 | Resets the progress bar and calls `end_query_internal`; attaches any end-of-query error to the result. |
| `ErrorData end_query_internal(bool success, bool invalidate_transaction)` | 121 | Tears down: **resets `sirius_active_query`** (closing the one-at-a-time window). Currently returns an empty `ErrorData` — no transaction management here yet. |

Supporting members:

| Function (signature) | Line | Role |
|---|---|---|
| `sirius_interface(ClientContext&, optional<string> query_label)` | 42 | Ctor; stores the context reference and an optional telemetry label. |
| `sirius_engine& get_sirius_engine()` | 243 | Accessor for `sirius_active_query->engine` (asserts a query is active). |
| `void check_executable_internal(PendingQueryResult&)` | 65 | Guards the execute phase: refuses a closed or errored pending result. |
| `void sirius_process_error(ErrorData&, const string& query) const` | 46 | Formats an error (JSON vs. location-annotated) per the connection setting. |
| `unique_ptr<T> sirius_error_result<T>(ErrorData, const string& query)` | 57 | **Error strategy:** wraps an error *into a result object* (`MaterializedQueryResult`/`PendingQueryResult`) instead of throwing — so the caller always gets a result it can check with `HasError()`. |
| `bind_prepared_statement_parameters(PreparedStatementData&, const PendingQueryParameters&)` *(free function)* | 35 | Binds statement parameters — currently a near-stub that binds an empty map (prepared-parameter support is not wired up yet). |

## The state machine to hold in your head

The whole file revolves around **one member**:
`unique_ptr<sirius_active_query_context> sirius_active_query` (header line 73). It's
the "a query is in flight" flag and bag of per-query state:

- **created** in `begin_query_internal` (83),
- **populated** in `sirius_pending_statement_internal` — gets the `engine`, then later
  the `sirius_prepared` and the open-result pointer,
- **read** everywhere via `get_sirius_engine()` and the `is_open_result` checks,
- **destroyed** in `end_query_internal` (125), via `cleanup_internal`.

Because there's exactly one of these, a `sirius_interface` runs **one query at a
time** (note the `D_ASSERT(!sirius_active_query)` at the start of
`begin_query_internal`). The `open_result` pointer inside it (header 53–61) is just a
sanity tether: it lets the code assert that the `PendingQueryResult` being executed is
the same one this query handed out.

## Things that will confuse you if you don't expect them

Since it's a *read*, here are the spots where the code looks like it's missing
something — it isn't, those paths just aren't built out yet:

- **`bind_prepared_statement_parameters` (35) does almost nothing** — it binds an empty
  parameter map. Sirius doesn't support prepared-statement parameters on the GPU path
  yet; the function exists to mirror DuckDB's shape.
- **`end_query_internal` (121) ignores its `success` / `invalidate_transaction` args**
  and returns an empty `ErrorData`. There's no transaction/rollback logic here — DuckDB
  owns the transaction; this is a deliberately thin stub.
- **`progress_bar` is reset but never really driven** — it's carried for parity with
  DuckDB's `ClientContext`, not yet a live feature.
- **Errors become results, not exceptions** (`sirius_error_result`). The two try/catch
  blocks (in `sirius_execute_query` and `sirius_execute_pending_query_result`) convert
  thrown exceptions into errored `MaterializedQueryResult`s; callers (e.g.
  `GPUExecutionFunction`) then check `HasError()` and decide whether to fall back to
  DuckDB. This is why the fallback logic lives *upstream* in `sirius_extension.cpp`,
  not here.

## Types fundamental to *this* file

The recurring DuckDB types here — `ClientContext`, `QueryResult`, `ErrorData`,
`PreparedStatementData` — are in
[`duckdb-types-glossary.md`](../reference/duckdb-types-glossary.md). The ones that are
specifically load-bearing for reading `sirius_interface.cpp`:

- **`sirius_active_query_context`** *(Sirius struct; header line 40)* — the per-query
  state holder described above: `query` string, `sirius_prepared`, the `engine`, the
  `progress_bar`, and the private `open_result` pointer. **Think:** the "current query
  in flight" record; its lifetime *is* the query's lifetime. Internalize this struct
  and the file falls into place.
- **`sirius_prepared_statement_data`** *(Sirius class; header line 26)* — the bundle
  passed *in* from the bind step: a DuckDB `PreparedStatementData` (result schema +
  values) plus the **Sirius physical plan** (`sirius_physical_operator` root). This is
  exactly what `GPUExecutionBind` built in the previous file. **Think:** "the prepared
  query" = its schema + its GPU plan, handed across the boundary.
- **`PendingQueryResult`** *(DuckDB type)* — the handle returned by the pending phase
  and consumed by the execute phase. It carries the result schema and an error slot,
  but holds *no rows yet*. Worth knowing well here because the whole two-phase model is
  built around it. **Think:** a "query that's set up but not run" — a ticket you redeem
  in the execute phase. *(Recurs in DuckDB generally; may get promoted to the shared
  glossary later.)*
- **`PendingQueryParameters`** *(DuckDB type)* — the parameter bundle threaded through
  every function. In practice unused on the GPU path today (see
  `bind_prepared_statement_parameters`), but it's in the signatures because the file
  mirrors DuckDB's. **Think:** "the bound `?` parameters" — present for parity, inert
  for now.

## Takeaway into the next file

`sirius_interface` is the **lifecycle conductor**: it opens per-query state, asks
`sirius_engine` to *build* pipelines (`initialize`, line 177), then to *run* them
(`execute`, line 197) and *yield* the result (`get_result`, line 102), then tears the
state down. It contains no GPU logic itself — every interesting verb delegates to
`sirius_engine`. That makes `src/sirius_engine.cpp` (execution-flow **Steps 4 & 5**)
the natural next read, and the place the real pipeline machinery begins.
