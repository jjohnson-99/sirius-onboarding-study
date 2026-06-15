# The DuckDB Extension API

Sirius is **not a fork of DuckDB** — it's a DuckDB **extension** (`sirius.duckdb_extension`)
that plugs in at well-defined seams. Understanding which seams it uses demystifies the
whole "doorway" layer: `sirius_extension.cpp`, the transparent optimizer hooks, and
`SiriusContext`. This is the framework behind those file-maps.

> **Prime around:** Week 2 (Days 1–3) — the doorway files
> ([`file-maps/sirius_extension.md`](../../file-maps/sirius_extension.md),
> [`file-maps/transparent/sirius_optimizer_extension.md`](../../file-maps/transparent/sirius_optimizer_extension.md),
> [`file-maps/sirius_context.md`](../../file-maps/sirius_context.md)) *are* extension-API code.

## What an extension is

A DuckDB extension is a shared library (`.duckdb_extension`) that DuckDB loads (`LOAD`)
and which **registers new functionality** into the running database. DuckDB is built to be
extended this way — `parquet`, `json`, `httpfs`, `icu` are all extensions. Sirius is one
too: it rides DuckDB's parser, optimizer, catalog, and transactions, and inserts GPU
execution at the extension points below — rather than reimplementing a SQL front end.

## The load entry point

Every extension has an entry function DuckDB calls at load time
(`DUCKDB_CPP_EXTENSION_ENTRY` → `LoadInternal` in Sirius). It receives an
`ExtensionLoader` / `DatabaseInstance` and **registers everything** in one place. (That's
Step 0 in [`file-maps/sirius_extension.md`](../../file-maps/sirius_extension.md).)
Loading a custom/unsigned build needs `allow_unsigned_extensions`.

## The extension points (and which Sirius uses)

| Extension point | What it adds | Sirius use |
|---|---|---|
| **Table function** | a `FROM`-clause function with the bind/execute contract ([`duckdb-table-functions.md`](duckdb-table-functions.md)) | `gpu_execution(...)`, `pin_table`, profiler fns |
| **Scalar / aggregate function** | new SQL functions | — (Sirius works at the operator level) |
| **Extension option (SET)** | `DBConfig::AddExtensionOption` | the `SET` knobs (`InitialGPUConfigs`) |
| **`OptimizerExtension`** | pre/post hooks around DuckDB's optimizer | **transparent execution** — capture the optimized plan |
| **`ClientContextState`** | per-connection state + query-lifecycle callbacks (`QueryBegin`/`QueryEnd`) and `OnFinalizePrepare` | **`SiriusContext`** — owns the GPU runtime *and* swaps in the GPU plan |
| **`ExtensionCallback`** | connection open/close, extension-load hooks | register `SiriusContext` on every connection |
| **Replacement scan** | rewrite a table name → a table function | the `read_parquet('s3://…')` routing |
| Parser / operator / storage / filesystem extensions | custom syntax, plan ops, storage, VFS | — (mostly not used) |

## The two seams that make Sirius work

Most of Sirius's "magic" is two extension points working together:

1. **Explicit path — a table function.** `CALL gpu_execution('SELECT …')` is just a
   registered table function; its bind generates the Sirius plan, its execute runs it. The
   simplest way an extension exposes new behavior.
2. **Transparent path — optimizer hook + plan rebind.** This is the clever one. An
   **`OptimizerExtension`** hook captures the optimized logical plan; then
   **`ClientContextState::OnFinalizePrepare`** (enabled by `CanRequestRebind()`) lets the
   extension **replace DuckDB's physical plan with its own** — a `PhysicalSiriusExecution`
   operator. So plain SQL transparently runs on the GPU, with **no new syntax**, by
   intercepting at these two seams. (Steps 1–2 in
   [`file-maps/transparent/sirius_optimizer_extension.md`](../../file-maps/transparent/sirius_optimizer_extension.md).)

Underneath both, **`ClientContextState`** gives Sirius a place to own its per-connection
GPU runtime with lifecycle hooks, and an **`ExtensionCallback`** registers that state on
each connection (see [`client-connections.md`](client-connections.md) for the
session/lifecycle model). The relevant DuckDB types —`ExtensionLoader`, `DatabaseInstance`,
`DBConfig`, `ClientContext` — are in
[`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md).

## Build / packaging (briefly)

Sirius builds through DuckDB's extension CMake infrastructure (`extension_config.cmake`,
the root `Makefile`), producing `sirius.duckdb_extension` (+ a loadable variant) that DuckDB
`LOAD`s. The extension links DuckDB as the host and registers itself via the entry point
above.

## The takeaway

The extension API is **the seam between DuckDB and Sirius**. Sirius doesn't own SQL parsing,
optimization, the catalog, or transactions — it hooks DuckDB at: *table functions*
(explicit API), *optimizer + rebind* (transparent execution), *client-context state*
(per-connection runtime), and *config options* (knobs). Read the doorway file-maps with
this table in hand and each one is "Sirius using extension point X."

## See also

- [`file-maps/sirius_extension.md`](../../file-maps/sirius_extension.md) — the registration code;
  [`file-maps/sirius_context.md`](../../file-maps/sirius_context.md) — `ClientContextState` +
  `OnFinalizePrepare`;
  [`file-maps/transparent/sirius_optimizer_extension.md`](../../file-maps/transparent/sirius_optimizer_extension.md)
  — the optimizer hooks.
- [`duckdb-table-functions.md`](duckdb-table-functions.md) — the bind/execute contract of
  the table-function point; [`client-connections.md`](client-connections.md) — the
  connection/lifecycle model the state hooks into.
