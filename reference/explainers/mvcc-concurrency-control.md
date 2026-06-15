# Concurrency Control & MVCC

When two connections touch the same data at the same time, *something* has to keep them
from seeing each other's half-finished work — that's **concurrency control**, and the
modern answer is **MVCC** (Multi-Version Concurrency Control). It's the mechanism that
makes the "multiple connections, concurrent" claim in
[`client-connections.md`](client-connections.md) actually safe.

> **Prime around:** Optional / background. The [onboarding plan](../../onboarding-path.md)
> deliberately *defers* OLTP storage-engine material (MVCC, WAL, B-trees) — Sirius
> doesn't implement any of it. Read this only if you're curious how concurrent DuckDB
> connections stay consistent; it pairs with `client-connections.md`.

## The problem: isolation

The **I** in ACID is *isolation*: concurrent transactions should each behave as if they
ran alone. Without it you get classic anomalies — a reader seeing another transaction's
uncommitted write (dirty read), or the same query returning different rows mid-query
because someone inserted in between. Concurrency control is how a database delivers
isolation while still letting transactions run at the same time.

## Two families: locking vs. MVCC

- **Lock-based (pessimistic).** Before reading/writing a row, take a lock; others wait.
  Correct, but readers block writers and writers block readers — contention kills
  throughput, especially when a long analytical read holds locks while writes pile up.
- **MVCC (optimistic, multi-version).** Don't overwrite data in place — keep **multiple
  versions** of each row. Every transaction reads from a **consistent snapshot** (the
  versions committed as of when it began). **Readers never block writers and writers
  never block readers**; conflicts are detected only between concurrent *writers*.

MVCC is what almost every modern engine (Postgres, DuckDB, Oracle, SQL Server's
snapshot mode) uses, because read-mostly workloads get near-lock-free reads.

## How MVCC works, concretely

Each row carries version metadata (which transaction created it, which deleted/superseded
it). A transaction gets a timestamp/ID at start; **visibility rules** then decide, for
each row version, "was this version committed before I started, and not yet superseded?"
— that's the snapshot it sees.

```
Row x = 10.
  Txn A (start ts=100) reads x        → sees 10
  Txn B (start ts=101) writes x = 20, commits
  Txn A reads x again                 → STILL sees 10   (its snapshot is frozen at ts=100)
  Txn C (start ts=102) reads x        → sees 20         (newer snapshot)
```

A keeps seeing `10` even after B commits — *snapshot isolation*. If A had *also* tried
to write `x`, the engine detects the write-write conflict at commit and aborts one of
them (optimistic: assume no conflict, check at commit, retry if wrong). Old versions are
reclaimed once no live snapshot can still see them (garbage collection / "vacuum").

DuckDB implements MVCC (serializable, in the HyPer / Neumann lineage — "Fast
Serializable MVCC"): in-process, multiple connections/threads can run transactions
concurrently, each on its own snapshot, with optimistic conflict checks at commit.

## Why an analytics engine still cares

OLAP is read-heavy, so you might expect concurrency control to matter less — and the
*write* contention story is indeed quieter. But you still want a long analytical scan to
run against a **stable snapshot** while data is being loaded or updated in another
connection, without either side blocking the other. MVCC delivers exactly that: the
dashboard query sees a consistent picture even as an ETL job appends rows.

## Where this lands in Sirius

Honest framing: **Sirius does not implement concurrency control at all** — it's read-only
GPU execution layered on DuckDB, so it inherits DuckDB's MVCC for the data it scans. A
Sirius query runs inside a DuckDB transaction, and the `DUCKDB_SCAN` reads the table at
that transaction's **snapshot** (the scan task holds a DuckDB scan state bound to the
snapshot's data blocks — the `DuckTableScanState`/`BlockHandle` lifetime the
`sirius_context.cpp` teardown comments fuss over). Once data is on the GPU it's an
**immutable** `cudf::table`, so there are no in-place updates to version — MVCC simply
doesn't arise on the GPU side.

Sirius's *own* concurrency problem is a different one: not transactional isolation, but
**GPU-resource scheduling** — serializing the shared GPU runtime across connections
(`query_lifecycle_mutex_`) and dispatching tasks across workers/GPUs (see
[`client-connections.md`](client-connections.md) and
[`file-maps/sirius_context.md`](../../file-maps/sirius_context.md)). So when you see
"concurrency" in Sirius, it means *thread/GPU scheduling*, not *MVCC* — those live in two
different worlds, and MVCC is firmly DuckDB's.

## See also

- [`client-connections.md`](client-connections.md) — connections/sessions and why
  Sirius serializes its shared runtime.
- [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) — `ClientContext`,
  `DatabaseInstance` (a transaction runs within a `ClientContext`).
- *Paper (deep, optional):* Neumann, Mühlbauer, Kemper, "Fast Serializable Multi-Version
  Concurrency Control for Main-Memory Database Systems" (2015).
