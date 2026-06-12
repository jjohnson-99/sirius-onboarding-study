# Sorting & ORDER BY

`ORDER BY` sounds trivial — put the rows in order — but sorting is one of the three
classic **blocking** operators (with join and aggregation), and doing it across many
threads/GPUs forces a clever trick that's worth knowing: you can't just sort the pieces
and glue them together. It's also where a *different* kind of partitioning shows up than
the one join and GROUP BY use.

> **Prime around:** Week 3 (the medium operator you read, `sirius_physical_top_n`, is
> ORDER + LIMIT) → Week 5 (the full 4-phase parallel sort appears in the pipeline
> splitting). See `docs/super-sirius/physical-plan-generation.md`.

## Why sorting blocks

To emit the *first* row of sorted output you must have seen the *smallest* key — which
could be the last row read. So a sort can't stream: it must consume **all** input before
producing anything. Like a hash-join build or an aggregation, it's a **pipeline breaker**
([`push-vs-pull-execution.md`](push-vs-pull-execution.md)). Cost is `O(n log n)` for a
comparison sort (less for radix/bucket sorts on fixed-width keys — see GPU below). Sorts
also carry detail: **ASC/DESC** per key, **NULLS FIRST/LAST**, and **stability** (do equal
keys keep their input order?).

## The parallel-sort problem

Say you split the input across 4 workers. Each sorts its chunk locally — but you **can't
concatenate** the four sorted chunks: worker 1's chunk might contain both the smallest and
the largest key. Independent local sorts don't compose the way partial *sums* do.

The fix is **sample sort** (a.k.a. range-partition sort):

1. **Local sort** — each worker/chunk sorts its piece.
2. **Sample** — sample keys across the data and compute `P-1` **boundary** values that cut
   the keyspace into `P` contiguous ranges of ~equal size.
3. **Range-partition** — redistribute rows so partition `i` holds exactly the keys in
   range `i`. Crucially, **partition `i` < partition `i+1`** in key order.
4. **Merge** — multi-way-merge the sorted runs within each partition. Now concatenating
   the partitions *in order* yields the globally sorted result.

```
chunks (locally sorted)        boundaries: [30, 70]
  [5,40,90]  [10,60,80] ──┐    range-partition by key
                          ├─▶  P0 (<30): [5,10]   ─merge▶ [5,10]      ┐
                          │    P1 (30–70):[40,60]  ─merge▶ [40,60]     ├▶ concat in order
                          └─▶  P2 (>70): [90,80]  ─merge▶ [80,90]     ┘  = [5,10,40,60,80,90]
```

### Range-partition vs. hash-partition (the key contrast)

Join and GROUP BY **hash-partition** — route equal keys to the same bucket, order
irrelevant ([`hash-join-build-probe.md`](hash-join-build-probe.md),
[`aggregation-and-group-by.md`](aggregation-and-group-by.md)). Sorting **range-partitions**
— route *ordered ranges* to buckets, so concatenating buckets preserves global order.
Same "redistribute, then combine" shape as the
[`morsel-driven`](morsel-driven-parallelism.md) local→merge pattern, but the partitioning
rule is chosen to fit the operation: *colocate* for join/agg, *order* for sort.

## TOP-N: don't sort if you only need a few

`ORDER BY … LIMIT N` does **not** need a full sort — you only need the N smallest/largest.
A **partial sort** (heap / `top-k` selection) finds them in ~`O(n log N)` and is far
cheaper than sort-then-truncate. It's a distinct operator (Sirius `TOP_N`), and it
parallelizes the same way: each chunk keeps its local top-N, then merge the per-chunk
top-Ns and keep the global top-N.

## GPU specifics

GPUs sort *very* well: for fixed-width keys, **radix sort** is effectively `O(n)` and
bandwidth-bound (no comparisons), which is why `cudf::sort` / `cudf::order_by` are fast.
Merging sorted runs uses `cudf::merge`.

## Sirius specifics

Sirius implements the sample-sort as a **4-phase pipeline split**
(`docs/super-sirius/physical-plan-generation.md`, `operators.md`):

| Phase | Operator | Does |
|---|---|---|
| 1 | `ORDER_BY` | local sort per batch (`gpu_order_impl::local_order_by` → `cudf::order_by`) |
| 2 | `SORT_SAMPLE` | sample N batches → compute `P-1` boundaries (custom `get_next_task_hint` waits for N samples) |
| 3 | `SORT_PARTITION` | range-partition by those boundaries |
| 4 | `MERGE_SORT` | multi-way merge runs (`gpu_merge_impl::merge_order_by`); FULL barrier |

And `TOP_N` → `MERGE_TOP_N`: local `cudf::top_k_order` then merge per-partition top-Ns.
The operator carries `orders` (keys + ASC/DESC + null ordering). This is the most
elaborate of the operator splits — a good capstone once you've seen the
join/aggregate two-phase shapes.

## See also

- [`hash-join-build-probe.md`](hash-join-build-probe.md) /
  [`aggregation-and-group-by.md`](aggregation-and-group-by.md) — the hash-partition
  siblings (vs sort's range-partition).
- [`morsel-driven-parallelism.md`](morsel-driven-parallelism.md) — the local→merge pattern
  underneath; [`push-vs-pull-execution.md`](push-vs-pull-execution.md) — sort as a pipeline
  breaker.
