# DuckDB Table Functions (bind & execute)

"Table function" is one of those DuckDB terms that's obvious once it clicks and
confusing until then. It comes straight out of the
[`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) bind→execute entries, and
it's the structural backbone of `sirius_extension.cpp`
([file-map](../../file-maps/sirius_extension.md)). Let's unpack it.

> **Prime around:** Week 2 (Days 1–2) — `sirius_extension.cpp` is wall-to-wall
> bind/execute pairs.

## Why "table"

SQL has three kinds of functions, distinguished by **what they return**:

| Kind | Returns | Used in | Example |
|---|---|---|---|
| **Scalar** | one value per row | `SELECT` list, `WHERE` | `upper(name)`, `abs(x)` |
| **Aggregate** | one value per group | with `GROUP BY` | `sum(x)`, `count(*)` |
| **Table** | a whole **table** (rows × columns) | the `FROM` clause | `read_parquet('f.parquet')`, `range(5)` |

A table function produces an entire **relation**, so you put it where a table goes —
in `FROM` — and select/filter/join it like any table:

```sql
SELECT * FROM range(5) WHERE range > 2;
SELECT l_quantity FROM read_parquet('lineitem.parquet') WHERE l_quantity > 10;
```

That's the whole reason for the name: its output *is* a table. (`CALL gpu_execution('...')`
is the same idea — `CALL` is just sugar for invoking a table function whose "table" is
the query's result set.)

## The two callbacks, and why two

To slot a table into a query, DuckDB needs two different things at two different times —
the same **plan-then-run** split you see all over Sirius:

**1. Bind — "describe the table." Runs once, at plan time, before any data.**
Reports the output **schema**: column names + types. DuckDB needs this *before*
execution so it can type-check and plan the surrounding SQL — to know
`SELECT l_quantity FROM read_parquet(...)` is valid, it must first learn the file has
an `l_quantity` column and its type. Bind fills two out-params:

```cpp
vector<LogicalType>& return_types;   // [BIGINT, DECIMAL, VARCHAR, ...]
vector<string>&      names;          // ["l_orderkey", "l_quantity", ...]
```

…and returns a `FunctionData` blob carrying any state execute will need (a file path, a
row count, a parsed plan…).

**2. Execute — "produce the rows." Runs repeatedly, at run time.**
DuckDB calls it over and over; each call fills one `DataChunk` (~2048 rows) and signals
exhaustion (cardinality 0). It gets the bind data back via `TableFunctionInput`.

So **bind answers "what does this table look like?" (schema, once); execute answers
"give me the next batch of rows" (data, many times).** Schema at plan time, data at run
time — the same separation as `sirius_interface`'s pending/execute and the engine's
initialize/execute.

> There's also an optional *init* callback that sets up global/local state for parallel
> scans, but bind + execute are the core.

## A minimal explicit example: `range(5)`

```sql
SELECT * FROM range(5);
--  range
--  -----
--    0
--    1
--    2
--    3
--    4
```

- **Bind:** "this produces **one column** named `range` of type **BIGINT**." → sets
  `return_types = {BIGINT}`, `names = {"range"}`, stashes `{start:0, stop:5}` as bind
  data. Now DuckDB knows the table's shape.
- **Execute:** emit rows `0..4` into a `DataChunk`, then return empty on the next call
  to say "done."

You never told `range` its schema — *it* told *you*, at bind time. That's the point of
bind.

## The example from the Sirius codebase

You met two real bind/execute pairs in `sirius_extension.cpp`. Registration makes the
pairing explicit — note the `TableFunction` constructor argument order:

```cpp
// name,           arg types,             EXECUTE callback,      BIND callback
TableFunction("gpu_execution", {LogicalType::VARCHAR}, GPUExecutionFunction, GPUExecutionBind);
```

- **`read_parquet('lineitem.parquet')`** → `SiriusReadParquetBind` (line 131) opens the
  parquet **footer only** (`describe_parquet`), reports column names/types, returns a
  `SiriusReadParquetBindData` with the URI + row count. Then (conceptually)
  `SiriusReadParquetFunction` streams the data rows. *Bind reads metadata; execute reads
  data* — a perfect illustration of the split: you learn a parquet file's schema without
  reading its contents (see [`parquet-format.md`](parquet-format.md) on why the footer
  makes that cheap).
- **`gpu_execution('SELECT ...')`** → `GPUExecutionBind` (line 510) parses the inner SQL
  to learn the **result schema**, builds the Sirius plan; `GPUExecutionFunction`
  (line 595) streams the result chunks out. The "table" here is the GPU query's result
  set.

## Tie-back to the glossary types

The four [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) types are exactly
the actors in this contract:

```
TableFunctionBindInput ──▶  BIND  ──▶ returns FunctionData (e.g. SiriusTableFunctionData)
   (the call's args)        (once)         │  declares return_types + names
                                           ▼
                          TableFunctionInput ──▶ EXECUTE ──▶ fills DataChunk
                          (carries bind_data back)  (repeatedly)   (rows out)
```

When the glossary says "bind runs once (determines the result schema and stashes
state), then execute runs repeatedly to emit rows," that's this schema-then-data,
plan-then-run split — and it underlies `gpu_execution`, the parquet scan, `pin_table`,
and DuckDB's own `read_parquet`/`range`.
