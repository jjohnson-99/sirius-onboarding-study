# Columnar vs. Row Storage (and why DuckDB is columnar)

Whether a database lays its data out by **row** or by **column** sounds like an
internal implementation detail — but for an analytics engine it's *the* foundational
choice, and it's why DuckDB, cuDF, and Sirius all look the way they do. It also
explains the `DataChunk`/`Vector` types in the
[`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) and the vectorized
execution model from [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md).

> **Prime around:** Week 1 — foundational background; read alongside the Week 1
> DB-theory readings, before tracing code.

## Same table, two physical layouts

Take a tiny table:

| id | name | qty |
|----|------|-----|
| 1  | alice | 10 |
| 2  | bob   | 20 |
| 3  | cara  | 30 |

**Row-major** (row storage) keeps each *row's* fields contiguous in memory/disk:

```
[1, alice, 10] [2, bob, 20] [3, cara, 30]
```

**Column-major** (columnar) keeps each *column's* values contiguous:

```
ids:   [1, 2, 3]
names: [alice, bob, cara]
qtys:  [10, 20, 30]
```

Logically identical table; physically opposite. The choice decides what's cheap.

## Why columnar wins for analytics

Analytics (OLAP) queries typically touch **few columns but many rows** —
`SELECT sum(l_quantity) FROM lineitem` reads **one** of `lineitem`'s ~16 columns, but
all ~6M rows. Three big consequences:

**1. You read only the columns you need (less I/O, better cache use).**
Columnar: `sum(l_quantity)` touches *only* the `l_quantity` array — a contiguous run of
bytes. Row-major: to reach each row's `l_quantity` you stride over the whole row,
dragging the other 15 columns' bytes through memory and cache. For wide tables and
narrow queries that's a 10×+ difference in bytes moved.

**2. Columns compress far better.**
A column holds values of **one type with similar distribution**, so general-purpose and
specialized encodings shine: run-length encoding, dictionary encoding, bit-packing,
delta, frame-of-reference. A row mixes an int, a string, and a decimal — much less
compressible. (This is exactly what [`parquet-format.md`](parquet-format.md) exploits on
disk.)

**3. Columns are the natural unit of vectorized / GPU execution.**
A column is a contiguous array, so an operator can run a tight loop over it — great for
CPU SIMD, and *essential* for GPUs, where thousands of threads each process one element
of the array in lockstep (SIMT). This is why `cudf::reduce(l_quantity_column, SUM)`
(the actual GPU work in
[`file-maps/op/sirius_physical_ungrouped_aggregate.md`](../../file-maps/op/sirius_physical_ungrouped_aggregate.md))
is one contiguous, massively-parallel pass. You *cannot* run an efficient GPU kernel
over row-major tuples; columnar is a prerequisite for the whole Sirius design.

## Why row storage still exists (OLTP)

Transactional (OLTP) workloads do the opposite: **few rows, all columns** — "fetch the
full order #12345," "insert one customer," "update this row's balance." Row storage puts
a whole record contiguously, so a point lookup or single-row insert touches one place.
Columnar would scatter that one row's fields across N separate column arrays — N writes
for one insert. So Postgres/MySQL (OLTP) are row stores; DuckDB/ClickHouse/Snowflake
(OLAP) are column stores. It's a workload trade-off, not a "better/worse."

| | Row store (OLTP) | Column store (OLAP) |
|---|---|---|
| Good at | point lookups, single-row insert/update | scans/aggregates over many rows, few columns |
| Reads | whole row cheaply | one column cheaply |
| Compression | modest | excellent (per-column) |
| Vectorize/GPU | awkward | natural |
| Examples | Postgres, MySQL | DuckDB, ClickHouse, cuDF/Sirius |

## DuckDB (and Sirius) specifically

DuckDB is columnar **both on disk and in memory**. Its in-memory unit is the
**`Vector`** — a contiguous array of one column's values — and a **`DataChunk`** is just
a bundle of column `Vector`s with a row count (~2048 rows). When the glossary calls a
`DataChunk` "the vector of vectorized execution," this is why: each operator processes a
chunk by looping over its column arrays, not over tuples.

Sirius takes columnar to its conclusion. A batch on the GPU is a `cudf::table` — a set
of `cudf::column`s, each a contiguous device array. Every operator's `execute()` gets
`table_view`s and calls a cuDF primitive that runs a kernel across whole columns. The
columnar layout is what makes GPU SQL possible at all; it's not an optimization bolted
on, it's the substrate.

## See also

- [`parquet-format.md`](parquet-format.md) — columnar *on disk*: how Parquet physically
  stores columns (and why Sirius reads it straight onto the GPU).
- [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md) — vectorized execution
  and the operator model that columnar enables.
- [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) — `DataChunk` / `Vector`.
