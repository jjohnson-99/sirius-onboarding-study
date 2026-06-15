# Database Instances, Connections & Config Scope

DuckDB has a tidy three-level hierarchy — **database → connection → query** — and
knowing which state lives at which level explains a surprising amount, including the
careful save-and-restore dance the transparent path does around DuckDB's optimizer
(which is what prompted this note). It's the scoping behind `DatabaseInstance`,
`ClientContext`, `DBConfig`, and `ClientConfig` in the
[`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md).

> **Prime around:** Week 2 (Day 3) — reading `sirius_context` and the transparent
> hooks; it's the scope model under `DBConfig` / `ClientContext` / `ClientConfig`.

## The three levels

| Level | Type(s) | Lifetime | One per… |
|---|---|---|---|
| **Database** | `DatabaseInstance`, `DBConfig` | the opened database | DB file / in-memory DB |
| **Connection / session** | `Connection` → `ClientContext`, `ClientConfig` | a client session | open connection |
| **Query** | (the active query context) | one statement | `SELECT`/`CALL` run |

Your mental model is exactly right: a `DatabaseInstance` *is* the database; a
**`Connection`** opens a session against it and gets a **`ClientContext`** (the
per-connection object every Sirius function takes). Multiple people — or the same
person in separate sessions — open multiple connections to the **one**
`DatabaseInstance`. Each connection runs its statements one at a time; different
connections can run concurrently (DuckDB handles the concurrency, MVCC, etc.).

So `ClientContext` = "this session," and it's the handle through which you reach both
the database (`context.db`, the shared `DatabaseInstance`) and the session's own state.

## Which config lives where

This is the crux of your question. Two config objects, two scopes:

- **`DBConfig` — database-wide.** Reached via `DBConfig::GetConfig(context)` (the
  context is just the path to it). Shared by *every* connection to the instance.
  Notably, **`DBConfig.options.disabled_optimizers` lives here** — it's instance-level,
  not per-connection.
- **`ClientConfig` — per-connection.** Hangs off `context.config`. Connection-scoped
  settings like `enable_optimizer`.

The subtlety the optimizer hooks expose: they reach `disabled_optimizers` *through* a
`ClientContext`, but the field they mutate is **database-wide** `DBConfig` state. Touch
it and you've changed config every connection sees — which is precisely why it must be
snapshotted and put back.

## The Sirius wrinkle: one shared runtime, serialized

DuckDB allows concurrent connections, but Sirius adds a twist worth knowing
(see [`file-maps/sirius_context.md`](../../file-maps/sirius_context.md)): the
**same `SiriusContext` instance is registered on every connection** of the database
(`OnConnectionOpened` installs the shared one), and the GPU query lifecycle is
**serialized** by `query_lifecycle_mutex_` so two connections can't run GPU queries —
or stomp on shared state like `disabled_optimizers` — at the same time. (Nested
*internal* connections, e.g. for Iceberg metadata, are excluded via
`InternalQueryGuard`.) That serialization is what makes a save/modify/restore of
database-wide config safe.

## Worked example: the optimizer disable/restore (your question)

The transparent path
([`file-maps/transparent/sirius_optimizer_extension.md`](../../file-maps/transparent/sirius_optimizer_extension.md))
needs DuckDB's optimizer to *not* run a few passes (`IN_CLAUSE`,
`COMPRESSED_MATERIALIZATION`, `STATISTICS_PROPAGATION`, `LATE_MATERIALIZATION`) that
produce plan shapes Sirius can't convert. So, for **one query**:

1. **Pre-optimizer hook** — snapshot the current `disabled_optimizers` (stored in
   `SiriusContext`, mutex-guarded), then add the four passes to it.
2. **DuckDB optimizes** — producing a plan shaped to be GPU-convertible.
3. **Post-optimizer hook** — **restore the snapshot immediately**, then capture the
   optimized logical plan.
4. *(later)* **`OnFinalizePrepare`** — try `create_plan()` on the captured plan. Works →
   GPU; throws → silent **CPU fallback**.

**Direct answer:** it's a **per-query** disable/restore, and the re-enable is
**unconditional** — it happens in step 3, *right after optimization*, **before** the
GPU-vs-CPU decision in step 4. It is **not** "only re-enabled when we fall back to
CPU." By the time a fallback could happen, the optimizers are already restored.

Two things make that robust:

- **Idempotent restore + a backstop.** `restore_transparent_disabled_optimizers` moves
  the snapshot out and clears it, so a second call is a no-op. `SiriusContext::QueryEnd`
  calls it *again* at the end of every query — a belt-and-suspenders restore in case the
  post-hook didn't run (e.g. optimization threw, or was skipped). So the disabled set is
  guaranteed back to normal by query end, no matter the path.
- **Why restore at all:** because `disabled_optimizers` is database-wide `DBConfig`
  state. Leaving it modified would degrade *later* CPU queries on the connection (and,
  absent serialization, other connections). The restore protects those **future**
  queries — not the current one.

One subtle consequence worth noticing: if this query *does* fall back to CPU, DuckDB
executes the plan it already built **with those four passes disabled** — still correct,
just missing a couple of optimizations. The disabling shapes the plan for GPU
convertibility; the restore exists purely so the *next* query starts clean.

## See also

- [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md) — `DatabaseInstance`,
  `ClientContext`, `DBConfig`, `ClientConfig`.
- [`file-maps/sirius_context.md`](../../file-maps/sirius_context.md) — the shared
  runtime, `InternalQueryGuard`, and the query-lifecycle serialization.
- [`file-maps/transparent/sirius_optimizer_extension.md`](../../file-maps/transparent/sirius_optimizer_extension.md)
  — the hooks in context (and the explicit path's matching `PrepareConnection` /
  `CleanupConnection` save-restore in
  [`file-maps/sirius_extension.md`](../../file-maps/sirius_extension.md)).
