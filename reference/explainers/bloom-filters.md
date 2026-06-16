# Bloom filters (and where they'd fit in Sirius)

A bloom filter is one of those structures that sounds exotic and turns out to be a bit array
plus a few hash functions — and in a database it earns its keep by answering one cheap
question: *"is this value definitely not in the set?"* That one-sided answer is exactly what
a join or a scan needs to throw away rows before paying to process them.

> **Prime around:** Week 5 — when you read the hash join and the dynamic-filter pushdown hook
> on it. Useful in Week 3 too (it's a *filter*). It is **not yet implemented** in Sirius;
> this explainer is half "what it is" and half "what exists today vs. what's planned."

## What a bloom filter is

A bloom filter tests **set membership** approximately, trading exactness for tiny space:

- A bit array of `m` bits, all 0 initially, and `k` independent hash functions.
- **Insert(x):** set the `k` bits at `h₁(x) … hₖ(x)`.
- **MayContain(x):** check those same `k` bits. If **any** is 0 → `x` is *definitely not*
  present. If **all** are 1 → `x` is *probably* present.

The asymmetry is the whole point:

| Answer | Meaning | Certainty |
|---|---|---|
| "not present" | x was never inserted | **always correct** (no false negatives) |
| "probably present" | x *might* be inserted, or it's a hash collision | may be a **false positive** |

You tune the **false-positive rate (FPR)** with `m` (bits) and `k` (hashes) relative to `n`
(items). Rule of thumb: ~**10 bits/item ≈ 1% FPR**. There is no `delete` (clearing bits would
create false negatives) — which is fine for the build-once-probe-many database use.

```
insert "orders" keys 7, 19         bitset (m=16, k=2)
  h(7)  = {2, 11}   →  ......1..1.....
  h(19) = {5, 11}   →  ..1..1...1.....   (bit 11 shared)

probe x=4 : bits {1, 9}  → bit 1 = 0  → DEFINITELY NOT in build → drop the row
probe x=7 : bits {2, 11} → both 1     → probably present → keep (verify in the real join)
```

## Why databases love them: prune before you pay

In analytics the expensive part is *touching* rows — decompressing, copying H2D, hashing,
materializing. A bloom filter lets a cheap upstream computation tell a downstream operator
*"don't bother with these"*:

- **Join sideways-information-passing (SIP) / dynamic join filters.** A hash join builds its
  hash table from the small side. While doing so it can build a bloom over the build keys and
  **push it to the probe side** (or into the scan feeding it). The probe/scan then drops rows
  whose key can't match *before* the join — often the single biggest win on a selective star-
  schema join. (This is the "dynamic" cousin of static predicate pushdown — see
  [filters & expression evaluation](filters-and-expression-evaluation.md) and
  [hash join build/probe](hash-join-build-probe.md).)
- **Parquet row-group / page skipping.** Parquet files can carry per-column-chunk bloom
  filters; a reader testing an equality predicate (`WHERE id = 42`) can skip whole row groups
  whose bloom says "not present" — complementary to min/max **zone-map** stats (which prune
  ranges) since bloom prunes *point* lookups that fall inside the min/max range.

Bloom is **runtime-only**: unlike a `col BETWEEN lo AND hi` zone map, "is this hash in this
bitset" has no SQL/AST expression — it needs a custom kernel (hash the column, gather bits,
AND). On a GPU the probe is **bandwidth-bound** (random loads into the bitset), which drives a
key heuristic below.

## How much of this is in Sirius? (Short answer: bloom isn't — yet)

Today **Sirius has no bloom filter**. What exists is the *generic dynamic-filter scaffolding*
bloom would eventually plug into, and even that is **not wired into the query path**. Concretely
(`src/op/sirius_dynamic_filter.{hpp,cpp}`):

| Piece | Status in code |
|---|---|
| `sirius_dynamic_filter` base (kind tag) + `sirius_ast_lowerable` capability mixin | **present** |
| `sirius_dynamic_filter_set` (thread-safe per-column channel: `push_filter` / `filters_for_column`) | **present** |
| `merge_ast_dynamic_filters_into_tree` (AND filters into a consumer's cuDF AST) | **present** |
| concrete filter kinds | **only `ZONE_MAP`** (`sirius_dynamic_zone_map_filter`) |
| `sirius_dynamic_filter_kind` enum | **`{ ZONE_MAP }`** — no `BLOOM`, no `IN_LIST` |
| a bloom filter class / GPU bloom build+probe kernel | **does not exist** |
| runtime-`apply` capability (bloom can't lower to AST, so it needs this) | **does not exist** (doc's Phase 1.2) |
| **producers / consumers actually using any of this** | **none** — grep the tree: nothing outside `sirius_dynamic_filter.cpp` references it |

So the framework is **scaffolding only**: the one concrete filter (zone-map) is AST-lowerable,
but no operator builds one and no operator consumes one — the whole subsystem is currently
dead in the query path. `docs/super-sirius/dynamic-filters.md` lays out the staged plan
(zone-map → **bloom (Phase 1.3)** → IN-list); bloom is **planned, not built**.

> **Don't confuse it with `op.filter_pushdown`.** The hash join carries a
> `duckdb::JoinFilterPushdownInfo` (`filter_pushdown`), threaded in by
> [`sirius_plan_comparison_join.md`](../../file-maps/planner/sirius_plan_comparison_join.md).
> That's **DuckDB's own** (static, min/max) join-filter-pushdown plumbing — a *different*
> mechanism from Sirius's `sirius_dynamic_filter` framework, and the dynamic side isn't active
> either. And Sirius's parquet **row-group pruning** uses min/max **statistics**
> (`filter_row_groups_with_stats`, see [parquet](parquet-format.md) /
> [`duckdb_scan_executor.md`](../../file-maps/op/scan/duckdb_scan_executor.md)), **not** parquet
> bloom filters.

### What a bloom implementation would need (per the design doc)

If/when Phase 1.3 lands, the shape is sketched in `dynamic-filters.md`:
- a **`sirius_dynamic_bloom_filter`** inheriting the (future) runtime-`apply` mixin, *not*
  `sirius_ast_lowerable` (no AST form);
- a **producer**: the hash-join build sizes a GPU bloom from build cardinality + a target FPR
  and pushes it into a `sirius_dynamic_filter_set`;
- a **consumer**: the probe input / parquet scan applies it post-decode;
- an **L2-cache sizing gate**: a bloom probe is bandwidth-bound, so the producer queries
  `cudaDeviceGetAttribute(cudaDevAttrL2CacheSize)` and **only builds the bloom if the bitset
  fits in L2** (else every probe pays full HBM bandwidth and it's not worth it) — falling back
  to zone-map/IN-list otherwise. A nice example of GPU-specific cost reasoning.

## Bloom vs. the filter Sirius *does* have

| | Zone map (min/max) — **exists** | Bloom — **planned** |
|---|---|---|
| Prunes | ranges (`col` outside `[min,max]`) | point membership (exact value not in set) |
| Best for | clustered / sorted data, range predicates | high-cardinality equality keys (joins) |
| AST-lowerable? | **yes** (`min ≤ col ≤ max`) | **no** (needs a runtime kernel) |
| In Sirius today | concrete class present, but unused | not implemented |

The two are complementary, which is why the doc stages them together: zone-map catches
out-of-range rows cheaply via AST; bloom catches in-range-but-absent keys via a kernel.

## Takeaway

A bloom filter is a small bit array + k hashes giving a one-sided "definitely-not / maybe"
membership test — perfect for pruning probe/scan rows from a join's build keys before paying to
process them. In Sirius it's **not implemented**: the dynamic-filter framework
(`sirius_dynamic_filter` base + channel + AST-merge helper) is in place but unused, the only
concrete kind is **zone-map** (not bloom), and there are no producers or consumers wired. Where
it *would* live is the hash-join build → probe/scan pushdown path
([comparison-join builder](../../file-maps/planner/sirius_plan_comparison_join.md),
[hash join](../../file-maps/op/sirius_physical_hash_join.md)); the staged plan is in
`docs/super-sirius/dynamic-filters.md`.
