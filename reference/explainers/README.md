# Explainers

Long-form answers to "things that are intuitive to people who know databases well" —
the background concepts behind Sirius/DuckDB that the file-maps and glossary assume but
don't teach. Each file is a standalone, question-driven explainer (one topic per file;
combined only when two ideas are genuinely one topic). Written for a strong-C++,
new-to-databases reader; cross-linked to the maps where the concept shows up in code.

| File | Answers |
|------|---------|
| [duckdb-table-functions.md](duckdb-table-functions.md) | What a DuckDB *table function* is, why it's called "table," and what the bind/execute two-callback contract means — with `range(5)` and the `gpu_execution`/`read_parquet` pairs. |
| [columnar-vs-row-storage.md](columnar-vs-row-storage.md) | Row-major vs column-major layout, why analytics engines (and GPUs) want columnar, why OLTP still uses rows, and why DuckDB/cuDF/Sirius are columnar to the core. |
| [vectorized-execution.md](vectorized-execution.md) | The processing model: Volcano vs. operator-at-a-time vs. vectorized (batch) execution, why ~2048, why it needs columnar, and how Sirius makes the "vector" a whole GPU column. |
| [parquet-format.md](parquet-format.md) | The Parquet file layout (row groups → column chunks → pages → footer), why the footer enables cheap schema + column pruning + row-group skipping, and how Sirius decodes it on the GPU. |

These are background reading, not tied to a specific onboarding week — pull one up
whenever a map references a concept you want grounded.
