# Explainers

Long-form answers to "things that are intuitive to people who know databases well" —
the background concepts behind Sirius/DuckDB that the file-maps and glossary assume but
don't teach. Each file is a standalone, question-driven explainer (one topic per file;
combined only when two ideas are genuinely one topic). Written for a strong-C++,
new-to-databases reader; cross-linked to the maps where the concept shows up in code.

Two ways to navigate them: the **reading thread** below (the *story* — concept builds on
concept) and the **index table** further down (the *schedule* — when to prime each, by
onboarding week).

## The "why GPU-native SQL" thread

These aren't a loose bag of topics — read in this order, they argue *why Sirius is shaped
the way it is*. Each step sets up the next:

1. **[Columnar vs. row storage](columnar-vs-row-storage.md)** — store data by column, not
   by row.
2. **[GPUs vs. CPUs for databases](gpu-vs-cpu-for-databases.md)** — *why columnar isn't
   optional*: a GPU only reaches its bandwidth over contiguous columns (memory
   coalescing). The hardware demands the layout.
3. **[Vectorized execution](vectorized-execution.md)** — process those columns in
   batches, not tuples; SIMT is this idea taken to the hardware limit.
4. **[Filters & expression evaluation](filters-and-expression-evaluation.md)** — how a
   single operator computes over a batch (FILTER/PROJECTION, and the predicates inside
   joins and aggregates).
5. **[Push vs. pull execution](push-vs-pull-execution.md)** — how batches are driven
   through the operator tree (push, within pipelines).
6. **[Morsel-driven parallelism](morsel-driven-parallelism.md)** — how those pipelines
   parallelize across cores and GPUs.
7. **[Hash join build/probe](hash-join-build-probe.md)** + **[Aggregation & GROUP BY](aggregation-and-group-by.md)**
   — the two heavyweight operators, built on the hashing/partitioning and combinable
   local→merge ideas from the steps above.

**Supporting concepts** sit beside the thread, pulled in where a map needs them rather
than as part of the arc: **[table functions](duckdb-table-functions.md)** (how queries and
data enter), **[Parquet](parquet-format.md)** (columnar *on disk*),
**[client connections](client-connections.md)** (sessions & config scope), and
**[MVCC](mvcc-concurrency-control.md)** (concurrency control — Sirius defers to DuckDB;
optional).

## Index — by when to prime

Ordered by **when to prime** (the week of [`onboarding-path.md`](onboarding-path.md)
where the concept first earns its keep). Each file repeats its own "Prime around" cue up
top. You'll also naturally reach for one whenever a map references a concept you want
grounded — the week is a *suggestion*, not a gate.

| File | Prime around | Answers |
|------|------|---------|
| [columnar-vs-row-storage.md](columnar-vs-row-storage.md) | Week 1 | Row-major vs column-major layout, why analytics engines (and GPUs) want columnar, why OLTP still uses rows, and why DuckDB/cuDF/Sirius are columnar to the core. |
| [vectorized-execution.md](vectorized-execution.md) | Week 1 | The processing model: Volcano vs. operator-at-a-time vs. vectorized (batch) execution, why ~2048, why it needs columnar, and how Sirius makes the "vector" a whole GPU column. |
| [gpu-vs-cpu-for-databases.md](gpu-vs-cpu-for-databases.md) | Week 1–2 | Why a GPU isn't just "more cores": throughput vs latency, SIMT warps, memory **coalescing** (why row-major kills GPU bandwidth), the PCIe boundary, and why bandwidth-bound DB ops win on GPUs — *if* the data is columnar. |
| [push-vs-pull-execution.md](push-vs-pull-execution.md) | Week 1–2 → 4 | Who drives data through the operator tree — demand-driven pull (Volcano) vs. data-driven push — the trade-offs, and why Sirius's ports/barriers/task-creator are exactly what a push-between-pipelines model needs. |
| [duckdb-table-functions.md](duckdb-table-functions.md) | Week 2 | What a DuckDB *table function* is, why it's called "table," and what the bind/execute two-callback contract means — with `range(5)` and the `gpu_execution`/`read_parquet` pairs. |
| [client-connections.md](client-connections.md) | Week 2 | The database → connection → query hierarchy, what's per-database (`DBConfig`) vs per-connection (`ClientConfig`), Sirius's shared/serialized runtime, and the per-query optimizer disable/restore lifecycle as a worked example. |
| [aggregation-and-group-by.md](aggregation-and-group-by.md) | Week 2 | Ungrouped vs grouped aggregates, hash vs sort grouping, the accumulator model, partial/combinable aggregation (local→merge), the distributive/algebraic/holistic classes (why AVG decomposes), and Sirius's two-phase cuDF implementation. |
| [filters-and-expression-evaluation.md](filters-and-expression-evaluation.md) | Week 3 | Expressions as ASTs; projection vs filter (predicate → mask → stream compaction); the materialize / AST-interpret / JIT-fuse strategies and why fusing matters on a bandwidth-bound GPU; filter pushdown; Sirius's `gpu_expression_executor` strategy knob. |
| [morsel-driven-parallelism.md](morsel-driven-parallelism.md) | Week 4 | How analytical engines spread work across cores/GPUs — morsels + work-stealing vs. plan-time exchange operators — and how it maps onto Sirius's task scheduler/creator. |
| [hash-join-build-probe.md](hash-join-build-probe.md) | Week 1 → 5 | Why hash join beats nested loops, the build (small side → hash table) / probe (stream big side) split, join types, partitioning when it won't fit, the build-side pipeline breaker, and Sirius's cuDF join modes. |
| [parquet-format.md](parquet-format.md) | Week 5 | The Parquet file layout (row groups → column chunks → pages → footer), why the footer enables cheap schema + column pruning + row-group skipping, and how Sirius decodes it on the GPU. |
| [mvcc-concurrency-control.md](mvcc-concurrency-control.md) | Optional | Concurrency control & MVCC — snapshot isolation, readers-don't-block-writers, why DuckDB uses it, and why Sirius *doesn't* (it inherits DuckDB's MVCC; its own "concurrency" is GPU scheduling). Background the plan defers. |
