# `op/sirius_physical_grouped_aggregate.cpp` → Operator Map

**The operator the Week 2 example query actually uses** — `SELECT l_returnflag,
SUM(l_quantity) … GROUP BY l_returnflag` → `HASH_GROUP_BY`. The plan had you read the
*ungrouped* sibling
([`file-maps/op/sirius_physical_ungrouped_aggregate.md`](sirius_physical_ungrouped_aggregate.md))
as a simpler stand-in; this map covers the real grouping path in full. Read with the
`HASH_GROUP_BY` / `MERGE_GROUP_BY` entries in `docs/super-sirius/operators.md` and the
`HASH_GROUP_BY` split in `docs/super-sirius/physical-plan-generation.md` (Part 3).

> Tagged **read** — it's the grouped aggregate the trace query uses; revisit at Weeks 4–5
> for the partition/merge + per-GPU pinning. Covers **two** files:
> `sirius_physical_grouped_aggregate.cpp` (HASH_GROUP_BY) and
> `sirius_physical_grouped_aggregate_merge.cpp` (MERGE_GROUP_BY). Source/doc paths relative to
> the Sirius repo root; lines as of 2026-06-14.

## The key idea: this is the GROUP BY operator

Where the ungrouped aggregate reduces a whole batch to **one** row (`cudf::reduce`, one global
group), the grouped aggregate buckets rows **by the GROUP BY key** and aggregates per bucket
via **`cudf::groupby`**. Same two-phase local→merge shape
([`reference/explainers/aggregation-and-group-by.md`](../../reference/explainers/aggregation-and-group-by.md)),
**plus** two things the ungrouped path lacks: a group **key** (`group_idx`) and a
**`PARTITION`-by-key** step so a given key's partials meet on one node before the merge.

## The three-operator split (plan-gen Part 3)

```
[scan … HASH_GROUP_BY]  ──FULL──▶  [PARTITION]  ──FULL──▶  [MERGE_GROUP_BY]  ──▶ downstream
   local groupby / batch            hash by key            re-group per partition
```

- **`HASH_GROUP_BY`** = `sirius_physical_grouped_aggregate` — local, per-batch (`cudf::groupby`).
- **`PARTITION`** = engine-injected; hash-partitions partial rows **by group key** so all
  partials for a key land in one partition (the "colocate equal keys" move shared with hash
  join — [`hash-join-build-probe.md`](../../reference/explainers/hash-join-build-probe.md)).
- **`MERGE_GROUP_BY`** = `sirius_physical_grouped_aggregate_merge` — combines a partition's
  partials into the final per-group rows (engine builds it via
  `construct_sirius_specific_operator`).

## Phase 1 — `HASH_GROUP_BY` (local, per batch)

| Function (signature) | Line | Role |
|---|---|---|
| ctor `sirius_physical_grouped_aggregate(types, expressions, groups, …)` | 28 / 51 | `convert_duckdb_aggregates_to_cudf(groups, expressions)` builds the cuDF plan: **`group_idx`** (which input columns are the GROUP BY keys), `cudf_aggregates` (`cudf::aggregation::Kind` per aggregate), `cudf_aggregate_idx`, `aggregate_slots` (AVG decomposition), `has_avg`, `has_count_distinct`. |
| `execute(input_data, stream)` | 74 | For each input batch, calls `gpu_aggregate_impl::local_grouped_aggregate(batch, group_idx, cudf_aggregates, …)` (line 84) → a partial grouped batch; returns the partials. |
| `is_source()` / `is_sink()` → `true` | hdr 97 / 105 | Blocking operator: sink of pipeline 1, source of its partial output. |

**Where grouping happens (read this):** `local_grouped_aggregate`
(`src/op/aggregate/gpu_aggregate_impl.cpp:192`) constructs a
`cudf::groupby::groupby` over the `group_idx` key columns and submits one
`aggregation_request` per aggregate, so within a batch every row is hashed by
`l_returnflag` and `l_quantity` is summed per group. COUNT(DISTINCT) uses a
`collect_set` aggregation here (finalized later in the merge).

## Phase 2 — `MERGE_GROUP_BY` (combine, per partition)

Unlike the ungrouped merge, this is a **`sirius_physical_partition_consumer_operator`** (it
consumes *partitioned* input):

| Function (signature) | Line | Role |
|---|---|---|
| ctor (clone-from-parent) | 54 | Copies the parent's cuDF defs (`group_idx`, `cudf_aggregates`, `aggregate_slots`, `has_avg`, `has_count_distinct`); sets `child_op`. |
| `get_next_task_input_data()` | 138 | **The partition-aware bit.** Under lock, drains **all** batches of **one** partition (`current_partition_index`), increments the index, and returns a `partitioned_operator_data` tagged with that partition id. The tag pins the task to `partition_idx % num_gpus` — because the merge materializes a **cuco hash table** to combine batches, so (like hash join) every task of a partition must stay on **one GPU**. |
| `execute(input_data, stream)` | 167 | (1) **Fast path** (167–181): single batch + no AVG/COUNT-DISTINCT → pass through. (2) **Merge** (184–193): `gpu_merge_impl::merge_grouped_aggregate(batches, num_group_cols, cudf_aggregates, …)` (line 188) re-groups the partition's partials by key via a hash table, applying each aggregate's combine kind. (3) **Finalize** (201–263): post-process AVG and COUNT(DISTINCT). |

**The finalize step (219–261)** — the part the ungrouped map only hints at:
- **AVG** (`slot.is_avg`): divide the merged `SUM` column by the merged `COUNT` column —
  DECIMAL via fixed-point `cudf::binary_operation(DIV)` to keep precision (231), else cast to
  FLOAT64 and divide (235–243).
- **COUNT(DISTINCT)** (`slot.is_count_distinct`): the merged column is a **LIST** (collected
  sets); `cudf::lists::count_elements` per row → the distinct count → cast to INT64 (251–254).
- otherwise the aggregate column is moved through zero-copy (259).

So grouping happens **twice**: `cudf::groupby` per batch (Phase 1), then a re-group across a
partition's batches in `merge_grouped_aggregate` (Phase 2), with `PARTITION` guaranteeing that
all of a key's partials sit in the same partition.

## Footguns

- **AVG / COUNT(DISTINCT)** are decomposed in Phase 1 (`collect_set`, SUM+COUNT) and only
  *finalized* in Phase 2 — don't expect a finished average out of `HASH_GROUP_BY`.
- DECIMAL widening + the **BIGINT-SUM CPU fallback** (int128 gap) apply here too
  ([`reference/explainers/types-duckdb-cudf-sirius.md`](../../reference/explainers/types-duckdb-cudf-sirius.md)).
- **Grouping sets** are carried (`grouping_sets`) but largely **TODO** (commented-out members,
  dead helpers noted at the top of the merge `.cpp`) — multi-set grouping isn't fully wired.
- **NULL group keys**: `cudf::groupby` is built with `null_policy::INCLUDE` (192), so NULL is
  its own group ([`reference/explainers/nulls-and-validity.md`](../../reference/explainers/nulls-and-validity.md)).

## Ungrouped vs grouped

| | Ungrouped | Grouped (this map) |
|---|---|---|
| SQL | `SUM(x)` (no GROUP BY) | `k, SUM(x) … GROUP BY k` |
| local op | `UNGROUPED_AGGREGATE` → `cudf::reduce` | `HASH_GROUP_BY` → **`cudf::groupby`** |
| keys | none (one global group) | `group_idx` (the GROUP BY columns) |
| **PARTITION step** | none | **yes** — hash-partition by key |
| merge op | `MERGE_AGGREGATE` (drains all batches) | `MERGE_GROUP_BY` (drains **one partition**; per-GPU-pinned, cuco hash table) |

## Types fundamental to *this* file

- **`cudf::groupby::groupby` + `groupby_aggregation`** *(cuDF)* — the GPU grouping primitive
  (in `gpu_aggregate_impl`); **Think:** "hash rows by the key columns, aggregate per bucket."
- **`group_idx` / `cudf_aggregates` / `AggregateSlot`** *(this file)* — the cuDF plan: key
  columns, per-aggregate kinds, and AVG/distinct decomposition metadata.
- **`sirius_physical_partition_consumer_operator`** *(Sirius base)* — base for operators that
  consume *partitioned* input one partition at a time (also MERGE_SORT, MERGE_TOP_N).
  **Think:** "process partition i's batches together, pinned to one GPU."
- **`gpu_merge_impl::merge_grouped_aggregate`** *(Sirius)* — the cross-batch re-group (hash
  table). **Think:** the Phase-2 combine.

## See also

- [`file-maps/op/sirius_physical_ungrouped_aggregate.md`](sirius_physical_ungrouped_aggregate.md)
  — the simpler sibling (shared local→merge plumbing).
- [`reference/explainers/aggregation-and-group-by.md`](../../reference/explainers/aggregation-and-group-by.md)
  — the *why*; [`weeks/week2-concepts.md`](../../weeks/week2-concepts.md) — Step 7 of the trace;
  [`reference/explainers/hash-join-build-probe.md`](../../reference/explainers/hash-join-build-probe.md)
  — the cuco-hash-table / one-partition-per-GPU pattern this shares.
