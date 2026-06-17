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
the way it is*. Each step sets up the next.

**The premise:** Sirius is an **[OLAP](oltp-vs-olap.md)** engine — analytics, not
transactions; every step below follows from that one fact.

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
   + **[Sorting & ORDER BY](sorting-and-order-by.md)** — the three heavyweight blocking
   operators, built on the partitioning and combinable local→merge ideas above
   (hash-partition for join/group-by; **range**-partition for sort).

**Supporting concepts** sit beside the thread, pulled in where a map needs them rather
than as part of the arc. *GPU systems* (how kernels run, are scheduled, and are fed):
**[GPU warps & execution model](gpu-warps-and-execution.md)** and
**[CUDA streams & async](cuda-streams-and-async.md)** (deeper dives on step 2's hardware),
plus **[GPU memory & spilling](gpu-memory-and-spilling.md)** (how Sirius copes with limited
GPU memory — reservations + tiered spill; Week 6 infrastructure), and **[NUMA](numa.md)**
(host-RAM locality on multi-socket boxes — why pinned staging memory is placed per node), and
**[cuDF](cudf.md)**
(the GPU primitive library the operators actually call), and **[cuCascade](cucascade.md)**
(the tiered-memory & data-flow library behind the batches/repositories), on
**[RMM](rmm.md)** (the GPU allocation layer beneath both); zoom out with
**[RAPIDS & the GPU data ecosystem](rapids-gpu-data-ecosystem.md)** (where these libraries
and Sirius sit). *DB context:*
**[DuckDB extension API](duckdb-extension-api.md)** (how Sirius plugs into DuckDB),
**[Substrait & plan IRs](substrait-and-plan-irs.md)** (how query plans are represented;
mostly legacy/vestigial in Sirius), **[table functions](duckdb-table-functions.md)** (how
queries and data enter),
**[Parquet](parquet-format.md)** (columnar *on disk*) + **[Iceberg](iceberg-table-format.md)**
(table format over it), **[client connections](client-connections.md)** (sessions & config
scope), and
**[MVCC](mvcc-concurrency-control.md)** (concurrency control — Sirius defers to DuckDB;
optional). *Execution & dev:* **[types (DuckDB↔cuDF↔Sirius)](types-duckdb-cudf-sirius.md)**,
**[NULLs & validity](nulls-and-validity.md)**, and **[testing Sirius](testing-sirius.md)**.
*Optimizations (mostly forward-looking in Sirius):*
**[bloom filters](bloom-filters.md)** (probabilistic membership for join/scan pruning — the
dynamic-filter framework exists as scaffolding, but bloom itself is not implemented).

## Index — by when to prime

Ordered by **when to prime** (the week of [`onboarding-path.md`](../../onboarding-path.md)
where the concept first earns its keep). Each file repeats its own "Prime around" cue up
top. You'll also naturally reach for one whenever a map references a concept you want
grounded — the week is a *suggestion*, not a gate.

| File | Prime around | Answers |
|------|------|---------|
| [oltp-vs-olap.md](oltp-vs-olap.md) | Week 1 | The premise: OLTP (many small txns, row store) vs OLAP (big read-mostly scans, column store), the contrast table, and why "Sirius is OLAP" is the root cause of columnar + vectorized + GPU. |
| [columnar-vs-row-storage.md](columnar-vs-row-storage.md) | Week 1 | Row-major vs column-major layout, why analytics engines (and GPUs) want columnar, why OLTP still uses rows, and why DuckDB/cuDF/Sirius are columnar to the core. |
| [vectorized-execution.md](vectorized-execution.md) | Week 1 | The processing model: Volcano vs. operator-at-a-time vs. vectorized (batch) execution, why ~2048, why it needs columnar, and how Sirius makes the "vector" a whole GPU column. |
| [gpu-vs-cpu-for-databases.md](gpu-vs-cpu-for-databases.md) | Week 1–2 | Why a GPU isn't just "more cores": throughput vs latency, SIMT warps, memory **coalescing** (why row-major kills GPU bandwidth), the PCIe boundary, and why bandwidth-bound DB ops win on GPUs — *if* the data is columnar. |
| [gpu-warps-and-execution.md](gpu-warps-and-execution.md) | Week 1–2 → 5 | The GPU execution model in depth: the thread/warp/block/grid/SM hierarchy, SIMT, latency hiding by warp-switching, occupancy, divergence, the memory hierarchy & bank conflicts, warp-level primitives — plus a terminology glossary. Deep-dive companion to the previous one. |
| [cuda-streams-and-async.md](cuda-streams-and-async.md) | Week 2 → 5–6 | The async model: streams as ordered queues, host/GPU concurrency, events & synchronization, copy/compute overlap (hiding PCIe), pinned memory, stream-ordered allocation (RMM) — and why every `execute()` takes a `cuda_stream_view`. |
| [gpu-memory-and-spilling.md](gpu-memory-and-spilling.md) | Week 6 | Why GPU memory is the hard constraint; RMM pooling; the GPU→host→disk tiers (cuCascade); reservations as admission control; the downgrade executor (spilling); read-only locks; OOM retry & CPU fallback; NUMA/multi-GPU spaces. |
| [numa.md](numa.md) | Week 5–6 | Non-Uniform Memory Access: why multi-socket servers have per-socket RAM (local fast / remote slow), why a bandwidth-bound DB co-locates thread+memory+GPU on one node, and Sirius's NUMA-aware pieces (`numa_small_pinned_mr`, per-node host spaces, `numa_id` copy hints, per-NUMA io_uring reactors). |
| [cudf.md](cudf.md) | Week 2 | What cuDF/libcudf is (RAPIDS GPU DataFrame library), `cudf::column`/`table`, the relational primitives it provides, and how Sirius operators are mostly thin translators from DuckDB plan nodes to `cudf::` calls. |
| [cucascade.md](cucascade.md) | Week 5 → 6 | The tiered-memory & data-flow library: `memory_space`/`Tier`, data representations, the `data_batch` (+ read-only locks), tier converters (spill/promote), and `shared_data_repository` — the residency counterpart to cuDF's compute. |
| [rmm.md](rmm.md) | Week 2 → 6 | RMM (RAPIDS Memory Manager): the GPU allocation floor — memory resources, the pool MR, stream-ordered allocation, device buffers, and the `rmm::cuda_stream_view` in every `execute()`. Beneath cuDF & cuCascade. |
| [rapids-gpu-data-ecosystem.md](rapids-gpu-data-ecosystem.md) | Week 1–2 | Orientation: what RAPIDS is (cuDF/RMM/cuML/Spark-RAPIDS…), Apache Arrow as the columnar glue, the composable-data-systems trend, and where Sirius (DuckDB front end + libcudf execution) and its GPU-SQL cousins fit. |
| [push-vs-pull-execution.md](push-vs-pull-execution.md) | Week 1–2 → 4 | Who drives data through the operator tree — demand-driven pull (Volcano) vs. data-driven push — the trade-offs, and why Sirius's ports/barriers/task-creator are exactly what a push-between-pipelines model needs. |
| [duckdb-extension-api.md](duckdb-extension-api.md) | Week 2 | How Sirius plugs into DuckDB (it's an extension, not a fork): the load entry point and the extension points — table functions, SET options, the `OptimizerExtension` + `ClientContextState` rebind that powers transparent execution, `ExtensionCallback`, replacement scans. |
| [substrait-and-plan-irs.md](substrait-and-plan-irs.md) | Week 2 | What a plan IR is (in-memory vs portable), what Substrait is (the open plan-interchange standard), and why Super Sirius translates DuckDB's `LogicalOperator` tree *directly* (Substrait being legacy/vestigial here). |
| [duckdb-table-functions.md](duckdb-table-functions.md) | Week 2 | What a DuckDB *table function* is, why it's called "table," and what the bind/execute two-callback contract means — with `range(5)` and the `gpu_execution`/`read_parquet` pairs. |
| [client-connections.md](client-connections.md) | Week 2 | The database → connection → query hierarchy, what's per-database (`DBConfig`) vs per-connection (`ClientConfig`), Sirius's shared/serialized runtime, and the per-query optimizer disable/restore lifecycle as a worked example. |
| [aggregation-and-group-by.md](aggregation-and-group-by.md) | Week 2 | Ungrouped vs grouped aggregates, hash vs sort grouping, the accumulator model, partial/combinable aggregation (local→merge), the distributive/algebraic/holistic classes (why AVG decomposes), and Sirius's two-phase cuDF implementation. |
| [filters-and-expression-evaluation.md](filters-and-expression-evaluation.md) | Week 3 | Expressions as ASTs; projection vs filter (predicate → mask → stream compaction); the materialize / AST-interpret / JIT-fuse strategies and why fusing matters on a bandwidth-bound GPU; filter pushdown; Sirius's `gpu_expression_executor` strategy knob. |
| [types-duckdb-cudf-sirius.md](types-duckdb-cudf-sirius.md) | Week 3 | The three type systems (DuckDB `LogicalType` ↔ `sirius::logical_type` ↔ `cudf::data_type`), supported types & fallbacks, DECIMAL/fixed-point, casting, and the int128/HUGEINT gap behind the BIGINT CPU fallbacks. |
| [nulls-and-validity.md](nulls-and-validity.md) | Week 3 | SQL three-valued logic (NULL = UNKNOWN) and its rules (WHERE, aggregates, joins), plus cuDF's out-of-band validity bitmask — how three-valued logic becomes mask arithmetic on the GPU. |
| [testing-sirius.md](testing-sirius.md) | Week 3 | The two test systems — SQLLogicTest (`.test` end-to-end SQL) and Catch2 C++ unit tests — their formats, how to run just yours, and the first-PR loop (build → test → pre-commit → PR). |
| [sorting-and-order-by.md](sorting-and-order-by.md) | Week 3 → 5 | Why sorting blocks; parallel sample sort (local sort → sample boundaries → **range**-partition → merge) vs the hash-partition of join/agg; TOP-N as partial sort; GPU radix sort; Sirius's 4-phase `ORDER_BY`/`SORT_SAMPLE`/`SORT_PARTITION`/`MERGE_SORT`. |
| [morsel-driven-parallelism.md](morsel-driven-parallelism.md) | Week 4 | How analytical engines spread work across cores/GPUs — morsels + work-stealing vs. plan-time exchange operators — and how it maps onto Sirius's task scheduler/creator. |
| [hash-join-build-probe.md](hash-join-build-probe.md) | Week 1 → 5 | Why hash join beats nested loops, the build (small side → hash table) / probe (stream big side) split, join types, partitioning when it won't fit, the build-side pipeline breaker, and Sirius's cuDF join modes. |
| [parquet-format.md](parquet-format.md) | Week 5 | The Parquet file layout (row groups → column chunks → pages → footer), why the footer enables cheap schema + column pruning + row-group skipping, and how Sirius decodes it on the GPU. |
| [iceberg-table-format.md](iceberg-table-format.md) | Week 5 | Apache Iceberg: a metadata/table layer over Parquet files (snapshots, time travel, schema/partition evolution, row-level deletes via delete files/deletion vectors) and how Sirius's `ICEBERG_SCAN` applies the deletes on the GPU. |
| [bloom-filters.md](bloom-filters.md) | Week 5 | What a bloom filter is (one-sided membership: "definitely not / maybe"), why DBs use them for join SIP / dynamic pushdown + parquet skipping, and an honest status check — **not implemented in Sirius**; the dynamic-filter scaffolding exists but is unused, and only zone-map (not bloom) is a concrete kind. |
| [mvcc-concurrency-control.md](mvcc-concurrency-control.md) | Optional | Concurrency control & MVCC — snapshot isolation, readers-don't-block-writers, why DuckDB uses it, and why Sirius *doesn't* (it inherits DuckDB's MVCC; its own "concurrency" is GPU scheduling). Background the plan defers. |
