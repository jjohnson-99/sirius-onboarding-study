# DuckDB Types Glossary

A shared reference for the DuckDB types that recur across the Sirius codebase. The
per-file maps in this folder link here instead of re-explaining these every time —
you'll meet most of them repeatedly and come to know them intimately, so they live in
one place.

Scope notes:
- **DuckDB types only.** Sirius-specific types (e.g. `sirius_interface`,
  `sirius_physical_operator`, `SiriusContext`) are documented in the per-file map
  where you first meet them, or — for the ubiquitous ones — in
  [`week1-concepts.md`](../weeks/week1-concepts.md).
- `unique_ptr<T>` / `shared_ptr<T>` are just `std::` ownership wrappers; `string` /
  `vector` are DuckDB aliases for the `std::` versions. Not glossary-worthy.
- Each entry gives what the type *is* and a **Think:** line for how to hold it
  mentally.

---

## The one to internalize first

- **`ClientContext`** — *the* central DuckDB object: the per-connection session a
  query runs on. Owns the transaction, the per-connection config, and
  `registered_state` (the keyed bag where Sirius stashes its `SiriusContext` under
  `"sirius_state"`). Almost every Sirius function that touches DuckDB takes a
  `ClientContext&`, because almost everything needs the session. **Think:** "the
  connection/session" — your handle to everything about the current query's
  environment.

## Engine & configuration

- **`DatabaseInstance`** — the running database engine for the process: owns the
  catalog, the global config, the connection list. One per opened database.
  **Think:** the engine/server object (vs. `ClientContext`, one per connection).
- **`DBConfig`** — database-wide configuration: registered extension options
  (`AddExtensionOption`), the disabled-optimizer set, the callback manager. Reached
  via `DBConfig::GetConfig(db)`. **Think:** the global knobs panel for the whole
  instance.
- **`ClientConfig`** — the per-connection counterpart of `DBConfig`, hanging off
  `ClientContext` (`context.config`). Holds connection-scoped settings like
  `enable_optimizer`. Code that temporarily changes optimizer behavior snapshots and
  restores this. **Think:** the per-connection knobs (vs. `DBConfig`'s global ones).

## Table-function plumbing (the bind → execute contract)

A DuckDB table function is two callbacks: **bind** runs once (determines the result
schema and stashes state), then **execute** runs repeatedly to emit result rows.
These types carry state between them — and this same contract underlies Sirius's
`gpu_execution`, the scan functions, `pin_table`, and more:

- **`TableFunctionBindInput`** — the parsed arguments to the call, handed to *bind*.
  Carries positional `inputs` and `named_parameters`, each already parsed into
  `Value`s. **Think:** the function's argument list.
- **`FunctionData`** — abstract base for bind-time state. *Bind* returns a
  `unique_ptr<FunctionData>`; *execute* receives it back (via `TableFunctionInput`).
  Subclass it to carry whatever execute needs. **Think:** the "state bundle" threaded
  from bind to execute.
- **`TableFunctionInput`** — what *execute* receives each call: a handle to the
  `bind_data` (cast back to the concrete `FunctionData` subclass) plus optional
  global/local state. **Think:** "here's the bind result again, now produce rows."
- **`DataChunk`** — DuckDB's columnar batch: a set of column `Vector`s plus a row
  count, capped at the vector size (~2048 rows). This *is* the "vector" of vectorized
  execution and the unit passed between operators; an execute callback fills an
  `output` `DataChunk&` with result rows (or sets cardinality 0 when done).
  **Think:** a batch of rows, stored column-major.
- **`LogicalType`** — a SQL column type (INTEGER, VARCHAR, DECIMAL, …). Bind declares
  the result schema by filling `vector<LogicalType>& return_types` alongside
  `vector<string>& names`. **Think:** a column's type tag; the two out-vectors
  together are the output table's header.

## Plans & results

- **`LogicalOperator`** — one node in DuckDB's *logical* query plan (relational
  algebra: GET, FILTER, JOIN, AGGREGATE…), the tree produced after parse+optimize and
  *before* physical planning. `unique_ptr<LogicalOperator>` is an owned plan/subtree.
  **Think:** the *what-to-compute* tree — the input Sirius translates into its own GPU
  physical plan.
- **`QueryResult`** — abstract base for a query's result (concretely a
  `MaterializedQueryResult` or a streaming variant). Exposes `Fetch()` (pull the next
  `DataChunk`) and `HasError()` / `GetError()`. **Think:** the result handle you pull
  batches from and check for errors.
- **`Connection`** — a DuckDB client connection with a `.Query(sql)` method — the same
  object a user program holds. Inside Sirius it's used to spin up internal connections
  (e.g. for CPU fallback). **Think:** a programmatic SQL client.

## Related types & idioms you'll see in passing

- **`Value`** — a single typed SQL value (the scalar in `inputs[0]`); `.ToString()`,
  `.GetValue<T>()`, `.IsNull()`. **Think:** one cell, type-tagged.
- **`PreparedStatementData`** — DuckDB's bundle of a prepared statement's result
  schema (`names`, `types`) and bound values. Sirius wraps one inside its own prepared
  data. **Think:** the schema/metadata half of a prepared query.
- **`OptimizerType`** — an enum naming individual optimizer passes (`IN_CLAUSE`,
  `COMPRESSED_MATERIALIZATION`, …); used in the disabled-optimizer set. **Think:** a
  switch label for one optimizer rule.
- **`D_ASSERT(cond)`** *(macro, not a type — `duckdb/common/assert.hpp`)* — a debug-time
  **invariant check**, all over the Sirius code (`D_ASSERT(sirius_active_query)`). Active
  only in `DEBUG` / `DUCKDB_FORCE_ASSERT` builds; **compiled to a no-op in release**
  (`NDEBUG`). When it trips it **throws `duckdb::InternalException`** ("Assertion
  triggered in file … line …: <cond>") — *not* `abort()` — so it unwinds like any
  exception (and is caught by Sirius's `try/catch` → errored result / fallback). **Think:**
  documented "this should never happen" assumption — enforced in debug/CI, absent (and *not*
  a safety net) in release. Recoverable/user errors use real exceptions
  (`InvalidInputException`, …) instead.
