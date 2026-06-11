# Explainers

Long-form answers to "things that are intuitive to people who know databases well" —
the background concepts behind Sirius/DuckDB that the file-maps and glossary assume but
don't teach. Each file is a standalone, question-driven explainer (one topic per file;
combined only when two ideas are genuinely one topic). Written for a strong-C++,
new-to-databases reader; cross-linked to the maps where the concept shows up in code.

Ordered roughly by **when to prime** (the week of [`onboarding-path.md`](onboarding-path.md)
where the concept first earns its keep). Each file repeats its own "Prime around" cue up
top. You'll also naturally reach for one whenever a map references a concept you want
grounded — the week is a *suggestion*, not a gate.

| File | Prime around | Answers |
|------|------|---------|
| [columnar-vs-row-storage.md](columnar-vs-row-storage.md) | Week 1 | Row-major vs column-major layout, why analytics engines (and GPUs) want columnar, why OLTP still uses rows, and why DuckDB/cuDF/Sirius are columnar to the core. |
| [vectorized-execution.md](vectorized-execution.md) | Week 1 | The processing model: Volcano vs. operator-at-a-time vs. vectorized (batch) execution, why ~2048, why it needs columnar, and how Sirius makes the "vector" a whole GPU column. |
| [push-vs-pull-execution.md](push-vs-pull-execution.md) | Week 1–2 → 4 | Who drives data through the operator tree — demand-driven pull (Volcano) vs. data-driven push — the trade-offs, and why Sirius's ports/barriers/task-creator are exactly what a push-between-pipelines model needs. |
| [duckdb-table-functions.md](duckdb-table-functions.md) | Week 2 | What a DuckDB *table function* is, why it's called "table," and what the bind/execute two-callback contract means — with `range(5)` and the `gpu_execution`/`read_parquet` pairs. |
| [client-connections.md](client-connections.md) | Week 2 | The database → connection → query hierarchy, what's per-database (`DBConfig`) vs per-connection (`ClientConfig`), Sirius's shared/serialized runtime, and the per-query optimizer disable/restore lifecycle as a worked example. |
| [morsel-driven-parallelism.md](morsel-driven-parallelism.md) | Week 4 | How analytical engines spread work across cores/GPUs — morsels + work-stealing vs. plan-time exchange operators — and how it maps onto Sirius's task scheduler/creator. |
| [parquet-format.md](parquet-format.md) | Week 5 | The Parquet file layout (row groups → column chunks → pages → footer), why the footer enables cheap schema + column pruning + row-group skipping, and how Sirius decodes it on the GPU. |
| [mvcc-concurrency-control.md](mvcc-concurrency-control.md) | Optional | Concurrency control & MVCC — snapshot isolation, readers-don't-block-writers, why DuckDB uses it, and why Sirius *doesn't* (it inherits DuckDB's MVCC; its own "concurrency" is GPU scheduling). Background the plan defers. |
