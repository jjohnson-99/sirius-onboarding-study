# Hash Joins: Build & Probe

A join asks "which rows of A match rows of B on a key?" — and the **hash join** answers
it without comparing every pair, which is why it's the workhorse join of analytical
engines. The build/probe split is also the cleanest example of a *pipeline breaker*
([`push-vs-pull-execution.md`](push-vs-pull-execution.md)) and a *shared concurrent sink*
([`morsel-driven-parallelism.md`](morsel-driven-parallelism.md)), and it's the shape of
Sirius's most complex operator.

> **Prime around:** Week 1 (the CMU joins lecture / [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md))
> for the concept; most useful **entering Week 5**, where `src/op/sirius_physical_hash_join.cpp`
> is the "study"-tagged operator.

## The baseline it beats: nested-loop join

The naive join is two nested loops: for every row of A, scan **all** of B looking for
matches. That's `O(|A| × |B|)` — fine for a handful of rows, catastrophic for millions.
Sirius keeps it only as a fallback (`NESTED_LOOP_JOIN`) for joins a hash join can't
express (pure inequality conditions).

## The hash join: two phases

Hash join trades that quadratic scan for two linear passes plus a hash table:

**1. Build phase.** Pick the **smaller** relation (the *build side*), read it **fully**,
hash each row on the join key, and insert into an in-memory hash table (`key → row(s)`).

**2. Probe phase.** Stream the **larger** relation (the *probe side*); for each row, hash
its key, look it up in the table, and emit matches.

```
BUILD (small side R)              PROBE (big side S, streamed)
  r1 (k=7) ─┐                       s: k=7 ─▶ hash(7) ─▶ found r1 ─▶ emit (s ⋈ r1)
  r2 (k=3)  ├─▶ hash table          s: k=9 ─▶ hash(9) ─▶ miss     ─▶ (drop, or null for OUTER)
  r3 (k=7) ─┘   { 3:[r2], 7:[r1,r3] }   s: k=3 ─▶ hash(3) ─▶ found r2 ─▶ emit (s ⋈ r2)
```

Cost drops to `O(|A| + |B|)` — each side is touched once, and hash lookups are ~`O(1)`.

### Why build the *small* side

The hash table should fit in memory (ideally cache). Building the smaller relation keeps
the table small; the probe side can be arbitrarily large because it's just **streamed**
past the table, never materialized. The optimizer estimates cardinalities to choose which
side is which. (In Sirius's planner the convention is **left = probe (streamed), right =
build (materialized)** — see `docs/super-sirius/physical-plan-generation.md`.)

### Join types ride on the same machinery

The build/probe structure is shared; the join *type* only changes what you emit on a
probe miss/match:

| Type | On probe match | On probe miss |
|---|---|---|
| INNER | emit joined row | drop |
| LEFT OUTER | emit joined row | emit probe row + NULLs for build cols |
| SEMI | emit probe row once | drop |
| ANTI | drop | emit probe row |

## The memory problem → partitioning

What if the build side doesn't fit in memory (or GPU memory)? **Partition first**:
hash-partition *both* sides into N buckets by the join key, so matching keys always land
in the same bucket pair, then hash-join bucket-by-bucket (each bucket's build side now
fits). This is the *grace / radix hash join*, and it's exactly why Sirius injects
`PARTITION` (and `CONCAT`) operators around joins — the GPU equivalent of slicing the
build/probe sides into GPU-sized pieces (`docs/super-sirius/physical-plan-generation.md`).

## The pipeline view: build is a breaker

You **cannot** probe until the build hash table is complete — so the build side is a
**pipeline breaker**. In Sirius terms the build pipeline feeds the join's `"build"` port
across a **FULL barrier**: the join task isn't ready until the build pipeline is finished
(that's the `get_next_task_hint` readiness logic and the `FULL`/`PARTIAL`/`PIPELINE`
barriers from [`push-vs-pull-execution.md`](push-vs-pull-execution.md)). The probe side
is its own (streaming) pipeline.

And under parallelism, all the build morsels insert into **one shared, concurrent hash
table** — the canonical "shared sink" of
[`morsel-driven-parallelism.md`](morsel-driven-parallelism.md) — then many probe morsels
look up against it in parallel.

## Sirius / GPU specifics (what Week 5 makes concrete)

`src/op/sirius_physical_hash_join.cpp` is the same build/probe shape, but the table is
built and probed **on the GPU** via cuDF primitives. From `docs/super-sirius/operators.md`,
three modes:

| Mode | When | cuDF |
|---|---|---|
| `STANDARD` | default, multi-partition | `cudf::inner_join` / `left_join` / `full_outer_join` |
| `BUILD_PROBE` | small build side (1 partition) | `cudf::hash_join` (build once, probe many) — or `cudf::distinct_hash_join` when build keys are *provably unique* |
| `MIXED_JOIN` | equality **and** inequality conditions | `cudf::mixed_join` (with a cuDF AST predicate) |

The build side is constructed as a child pipeline whose sink is the join's build port
(`build_join_pipelines`), partitioned to fit GPU memory; the probe side streams against
it. Pure-inequality joins (no equality key to hash) fall back to `NESTED_LOOP_JOIN`.
So when you reach the Week 5 file-map, every piece of it — modes, the `"build"` port, the
PARTITION/CONCAT wrapping, the distinct-keys optimization — is a GPU realization of the
build/probe story here.

## See also

- [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md) — hash join in the Week 1
  vocabulary (and how Sirius implements it), alongside physical plan / operator /
  pipeline.
- [`push-vs-pull-execution.md`](push-vs-pull-execution.md) — pipeline breakers and the
  FULL barrier the build side uses.
- [`morsel-driven-parallelism.md`](morsel-driven-parallelism.md) — the shared concurrent
  hash table as the parallel "sink."
