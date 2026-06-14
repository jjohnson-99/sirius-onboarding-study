# `op/sirius_physical_grouped_aggregate.cpp` → Operator Map (stub)

**This is the operator the Week 2 example query actually uses** — `SELECT l_returnflag,
SUM(l_quantity) … GROUP BY l_returnflag` → `HASH_GROUP_BY`. The Week 2 plan had you read the
*ungrouped* sibling ([`file-maps/op/sirius_physical_ungrouped_aggregate.md`](file-maps/op/sirius_physical_ungrouped_aggregate.md))
as a simpler stand-in; this stub fills the gap: where the actual **grouping** happens.

> **Stub:** grounded in the operator + `docs/super-sirius/operators.md` /
> `physical-plan-generation.md`, but lighter than a read-closely map (the `MERGE_GROUP_BY`
> internals aren't walked). Read the ungrouped map first for the shared plumbing.
>
> **Prime around:** Week 2 (it's what the trace query uses) → revisit Weeks 4–5 (the
> `HASH_GROUP_BY → PARTITION → MERGE_GROUP_BY` pipeline split). Source/doc paths are relative
> to the Sirius repo root; lines as of 2026-06-14.

## The key idea: this is the GROUP BY operator

Where the ungrouped aggregate reduces a whole batch to **one** row (`cudf::reduce`, one global
group), the grouped aggregate buckets rows **by the GROUP BY key** and aggregates per bucket —
via **`cudf::groupby`**. Same two-phase local→merge shape (see
[`reference/explainers/aggregation-and-group-by.md`](reference/explainers/aggregation-and-group-by.md)),
**plus** a key: grouping, and a `PARTITION`-by-key step to bring matching keys together across
batches.

## The three-operator split (from `physical-plan-generation.md`)

```
[scan … HASH_GROUP_BY]  ──FULL──▶  [PARTITION]  ──FULL──▶  [MERGE_GROUP_BY]  ──▶ downstream
   local groupby / batch            hash by key            combine per group
```

- **`HASH_GROUP_BY`** = `sirius_physical_grouped_aggregate` (this file) — local, per-batch.
- **`PARTITION`** = hash-partition the partial rows **by group key** (the engine injects it).
- **`MERGE_GROUP_BY`** = `sirius_physical_grouped_aggregate_merge.cpp` — finalize per group
  (drains one partition per task, like `MERGE_SORT`; engine builds it via
  `construct_sirius_specific_operator`).

## Functions (the grouped operator)

| Function (signature) | Line | Role |
|---|---|---|
| ctor `sirius_physical_grouped_aggregate(types, expressions, groups, …)` | 28 / 51 | `convert_duckdb_aggregates_to_cudf(groups, expressions)` fills the cuDF plan: **`group_idx`** (which input columns are GROUP BY keys), `cudf_aggregates` (`cudf::aggregation::Kind` per aggregate), `cudf_aggregate_idx`, `aggregate_slots` (AVG decomposition), `has_avg`, `has_count_distinct`. |
| `execute(input_data, stream)` | 74 | **Where grouping happens.** For each input batch, calls `gpu_aggregate_impl::local_grouped_aggregate(batch, group_idx, cudf_aggregates, …)` (line 84) — the `cudf::groupby` that buckets rows by the `group_idx` columns and aggregates per group → a partial grouped batch. Returns the partials. |
| `is_source()` / `is_sink()` → `true` | hdr 97 / 105 | A blocking operator: sink (accumulates) of pipeline 1, source of the result it emits. |

`group_idx` is the giveaway versus the ungrouped operator: it names the **key columns**.
`gpu_aggregate_impl::local_grouped_aggregate` (in `src/op/aggregate/`) is the cuDF wrapper;
this operator is the thin DuckDB-facing shell around it.

## Footguns (same family as ungrouped)

- **AVG** → decomposed into SUM + COUNT (`has_avg`, `aggregate_slots`).
- **COUNT(DISTINCT)** → `COLLECT_SET` then count uniques (`has_count_distinct`).
- DECIMAL widening / the BIGINT-SUM CPU fallback — same int128 story
  ([`reference/explainers/types-duckdb-cudf-sirius.md`](reference/explainers/types-duckdb-cudf-sirius.md)).
- **Grouping sets** are carried (`grouping_sets`) but largely TODO (commented-out members) —
  multi-set grouping isn't fully wired.

## Ungrouped vs grouped (the contrast)

| | Ungrouped (you read) | Grouped (this file) |
|---|---|---|
| SQL | `SUM(x)` (no GROUP BY) | `k, SUM(x) … GROUP BY k` |
| local op | `UNGROUPED_AGGREGATE` → `cudf::reduce` | `HASH_GROUP_BY` → **`cudf::groupby`** |
| keys | none (one global group) | `group_idx` (the GROUP BY columns) |
| **PARTITION step** | none | **yes** — hash-partition by key |
| merge op | `MERGE_AGGREGATE` | `MERGE_GROUP_BY` |

## See also

- [`file-maps/op/sirius_physical_ungrouped_aggregate.md`](file-maps/op/sirius_physical_ungrouped_aggregate.md)
  — the simpler sibling (shared two-phase plumbing).
- [`reference/explainers/aggregation-and-group-by.md`](reference/explainers/aggregation-and-group-by.md)
  — the *why* (hash grouping, accumulators, distributive/algebraic/holistic);
  [`weeks/week2-concepts.md`](weeks/week2-concepts.md) — Step 7 of the trace.
