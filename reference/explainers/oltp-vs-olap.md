# OLTP vs. OLAP

This is the single distinction that explains *why Sirius (and DuckDB) look the way they
do*. Almost every choice in the [reading thread](README.md) — columnar storage,
vectorized execution, betting on GPU bandwidth — is downstream of one fact: **Sirius is an
OLAP engine, not an OLTP one.** Here's the difference, refreshed.

> **Prime around:** Week 1 — it's the framing for everything else; read it first and the
> rest of the thread reads as "consequences of being OLAP."

## Two workload archetypes

- **OLTP — Online Transaction Processing.** The database behind an *application*: many
  tiny transactions, mostly point operations. "Fetch order #12345," "insert a customer,"
  "decrement this balance." High write concurrency, each transaction touches **few rows
  but whole records**, and latency-per-transaction (milliseconds) is what matters.
  *Postgres, MySQL, SQLite.*
- **OLAP — Online Analytical Processing.** The database behind *analytics / reporting*:
  few but **big** read-mostly queries. "Total revenue by region last quarter" scans
  millions of rows but touches **few columns** and aggregates. Throughput over huge scans
  is what matters. *DuckDB, ClickHouse, Snowflake, BigQuery — and Sirius.*

## The contrast, and what each choice implies

| | OLTP | OLAP |
|---|---|---|
| Query shape | many small txns; point lookups, inserts/updates | few large scans + aggregates/joins |
| Read/write | write-heavy, mixed | read-mostly |
| Rows per query | few | millions |
| Columns per query | the **whole row** | a **few** of many |
| Best storage | **row** store (whole record contiguous) | **column** store ([`columnar-vs-row-storage.md`](columnar-vs-row-storage.md)) |
| Indexing | B-tree indexes for point access | scan + compression + min/max skipping |
| Concurrency | high; needs **MVCC** ([`mvcc-concurrency-control.md`](mvcc-concurrency-control.md)) | read-mostly; concurrency matters less |
| Data model | normalized | denormalized / star schema |
| Bottleneck | latency, lock contention | **memory bandwidth** over big scans |
| Goal | ms per transaction | throughput per query |

Read the table top-to-bottom and the architecture falls out: because OLAP queries scan
many rows but few columns, you store **by column**; because columns are contiguous
same-type arrays, you can **compress** them and process them **vectorized**; because the
work is **bandwidth-bound**, a **GPU**'s huge bandwidth wins
([`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md)) — *if* the layout is
columnar. OLTP pushes the opposite way: whole-row access and single-row writes favor a row
store with B-trees and MVCC. It's a workload trade-off, not "better vs. worse."

> Some systems chase **HTAP** (Hybrid Transactional/Analytical Processing) — both at once,
> usually with separate row and column representations under one engine. Most engines still
> specialize.

## Where Sirius lands

Sirius is squarely OLAP — in fact a *specialist within OLAP*: it accelerates the
read-only analytical operators (scan, filter, project, join, aggregate, sort) on the GPU,
and leans on **DuckDB** for everything OLTP-adjacent it doesn't do — transactions, MVCC,
storage, writes. That division is exactly why the explainers split the way they do: the
[reading thread](README.md) is the *OLAP execution* story; the supporting
[`mvcc-concurrency-control.md`](mvcc-concurrency-control.md) is the *OLTP-style concurrency*
that Sirius deliberately leaves to DuckDB. So "Sirius is an OLAP engine" isn't a label —
it's the root cause of the entire design.

## See also

- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the storage consequence of
  being OLAP.
- [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) — why OLAP's bandwidth-bound
  scans suit a GPU.
- [`mvcc-concurrency-control.md`](mvcc-concurrency-control.md) — the OLTP-flavored
  concurrency Sirius inherits rather than implements.
