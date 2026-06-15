# Aggregation & GROUP BY

Aggregation collapses many rows into few — a `SUM`, a `COUNT`, one number per group — and
`GROUP BY` is just "do that aggregation once per distinct key." Simple to state, but *how*
you do it on parallel hardware shapes a lot of the engine, and it's the operation that
closes the Week 2 trace.

> **Prime around:** Week 2 — the aggregate is the "closes the loop" operator
> ([`file-maps/op/sirius_physical_ungrouped_aggregate.md`](../../file-maps/op/sirius_physical_ungrouped_aggregate.md)).
> The grouped variant + merge split deepens in Weeks 4–5.

## Two flavors

- **Ungrouped (scalar) aggregate** — whole table → **one** row.
  `SELECT SUM(l_quantity) FROM lineitem`.
- **Grouped aggregate** — **one row per distinct key**.
  `SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY l_returnflag`.

Grouped is the general case; ungrouped is "everything is one big group." Both compute the
same aggregate functions (`SUM`, `COUNT`, `MIN`, `MAX`, `AVG`, …) — each reduces a column
to a value.

## How grouping is done: hash vs. sort

Two classic strategies (the CMU aggregation lecture):

- **Hash aggregation.** Keep a hash table keyed by the group key, where each entry holds
  that group's **running accumulator**. For each input row: hash the key, find-or-create
  the group, update its accumulator (`sum += x`, `count += 1`, `min = min(min, x)`).
  `O(n)`, no ordering. This reuses the hashing machinery from
  [`hash-join-build-probe.md`](hash-join-build-probe.md). It's DuckDB's and Sirius's
  default (`HASH_GROUP_BY`).
- **Sort aggregation.** Sort by the group key, then sweep: consecutive equal keys form a
  group. `O(n log n)`, but gives sorted output and bounded memory — used when output needs
  to be ordered or the hash table won't fit.

The mental model either way: **each group has an accumulator; one pass folds rows into
accumulators.**

## Aggregation is a pipeline breaker

You can't emit a group's result until you've seen **all** its rows — so aggregation, like
a hash-join build, is **blocking**: accumulate everything, then emit. In Sirius terms the
aggregate sits behind a **FULL barrier**
([`push-vs-pull-execution.md`](push-vs-pull-execution.md)); downstream waits for it to
finish.

## The parallelism trick: partial (combinable) aggregation

Here's the idea that drives the whole implementation. Most aggregates are **decomposable**:
compute *partial* aggregates on subsets, then **combine** the partials.

```
SUM(all)   = SUM(partial SUMs)
COUNT(all) = SUM(partial COUNTs)
MIN(all)   = MIN(partial MINs)
MAX(all)   = MAX(partial MAXs)
```

So each thread/morsel aggregates its slice **locally**, and a **merge** step combines the
locals — the same local→merge, "shared combinable sink" pattern as
[`morsel-driven-parallelism.md`](morsel-driven-parallelism.md). This is exactly Sirius's
two-phase shape:

- ungrouped: **`UNGROUPED_AGGREGATE` (local) → `MERGE_AGGREGATE` (combine)**
- grouped: **`HASH_GROUP_BY` (local) → `PARTITION` → `MERGE_GROUP_BY` (combine)**

Why the extra `PARTITION` for grouped? Because the *same key can appear in different
batches*. Hash-partitioning by the group key routes all rows of a given key to one
partition, so the merge can finalize each group in one place. (Ungrouped needs no
partition — there's a single global group.) It's the same "bring matching keys together"
move as grace-hash-join partitioning.

### The three aggregate classes (why AVG is special)

Not everything combines the same way (the data-cube taxonomy):

| Class | Combine rule | Examples |
|---|---|---|
| **Distributive** | combine partials directly | `SUM`, `COUNT`, `MIN`, `MAX` |
| **Algebraic** | combine via a bounded set of distributive sub-aggregates | `AVG`, `STDDEV` |
| **Holistic** | not finitely decomposable — needs all values | `MEDIAN`, `COUNT(DISTINCT)` |

**`AVG` is the canonical algebraic case:** you can't average averages. So it's decomposed
into `SUM` + `COUNT` (both distributive), those are merged separately, and the division
happens at the very end. That decomposition is precisely what the
[`ungrouped-aggregate file-map`](../../file-maps/op/sirius_physical_ungrouped_aggregate.md)
shows in code. `COUNT(DISTINCT)` is holistic — Sirius handles it specially (a
`COLLECT_SET` aggregation, then counting uniques) rather than a simple combine.

## A worked example

`SELECT k, SUM(x) FROM t GROUP BY k`, with `t` split across two batches:

```
batch 1: (a,10) (b,3) (a,5)     ── local groupby ─▶ { a:15, b:3 }
batch 2: (a,2)  (b,7)           ── local groupby ─▶ { a:2,  b:7 }
                                       partition by key + merge ─▶ { a:17, b:10 }
```

Each batch aggregates locally; the merge sums the partial sums per key. Two linear passes,
fully parallel.

## Sirius / GPU specifics (Week 2, deepened in 4–5)

- **Ungrouped:** local `cudf::reduce` per batch produces a 1-row partial; `MERGE_AGGREGATE`
  combines partials (`docs/super-sirius/operators.md`).
- **Grouped:** local `cudf::groupby` per batch; `PARTITION` (hash by key) then
  `MERGE_GROUP_BY` finalizes (`docs/super-sirius/physical-plan-generation.md`).
- **GPU footguns** (honest, from the file-map / operators.md): `AVG` → `SUM` +
  `COUNT_VALID`; `COUNT(DISTINCT)` → `COLLECT_SET`; HUGEINT downcast to BIGINT; and
  **BIGINT `SUM` falls back to CPU** because the GPU lacks an INT128 accumulator (silent
  overflow otherwise). These are exactly the kinds of correctness landmines the file-map
  flags.

## See also

- [`file-maps/op/sirius_physical_ungrouped_aggregate.md`](../../file-maps/op/sirius_physical_ungrouped_aggregate.md)
  — the two-phase aggregate in code (the *how* to this *why*).
- [`weeks/week2-concepts.md`](../../weeks/week2-concepts.md) — where the aggregate closes the
  end-to-end trace.
- [`hash-join-build-probe.md`](hash-join-build-probe.md) — the hashing/partitioning this
  shares; [`morsel-driven-parallelism.md`](morsel-driven-parallelism.md) — the combinable
  local→merge pattern.
