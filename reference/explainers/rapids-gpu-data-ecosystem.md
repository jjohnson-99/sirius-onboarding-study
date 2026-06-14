# RAPIDS & the GPU Data Ecosystem

cuDF ([`cudf.md`](cudf.md)) and RMM ([`rmm.md`](rmm.md)) aren't Sirius inventions — they're
part of **RAPIDS**, NVIDIA's open-source GPU data-science suite, and Sirius sits inside a
broader "composable GPU data" ecosystem. This is the zoom-out that frames the library
explainers: what RAPIDS is, what else is in it, and where a GPU SQL engine like Sirius fits.

> **Prime around:** Week 1–2 (orientation) — read once to place cuDF/RMM in context; not
> needed for any specific code-reading task.

## What RAPIDS is

**RAPIDS** is a family of GPU-accelerated libraries that mirror the popular CPU "PyData"
data-science stack, so analysts get GPU speed with familiar APIs. All share a foundation:
**CUDA** (compute), **RMM** (memory), and an **Apache Arrow-style columnar** memory layout.

| CPU world | RAPIDS (GPU) | What it does |
|---|---|---|
| pandas | **cuDF** | DataFrames / columnar analytics (libcudf is the C++ core) — **Sirius uses this** |
| (allocator) | **RMM** | GPU memory management — **Sirius uses this** |
| scikit-learn | **cuML** | machine learning |
| NetworkX | **cuGraph** | graph analytics |
| NumPy | **CuPy** (adjacent) | n-dim arrays |
| Spark SQL | **Spark RAPIDS** | offloads Spark query plans to GPU via libcudf |
| — | **RAFT / cuVS** | shared primitives, vector search |

Sirius depends on the bottom two (**cuDF/libcudf** + **RMM**). cuCascade
([`cucascade.md`](cucascade.md)) is a *separate* NVIDIA library (tiered memory) built on RMM
— complementary to RAPIDS rather than branded part of it.

## The shared foundation: Apache Arrow

The glue under all of this is **Apache Arrow** — an open, language-agnostic **columnar
in-memory format**. Because cuDF columns are Arrow-compatible, data moves between tools
(pandas, Polars, DuckDB, Spark, Parquet files, cuDF) with little or no conversion, often
**zero-copy**. Arrow is why "columnar" ([`columnar-vs-row-storage.md`](columnar-vs-row-storage.md))
is an *ecosystem* choice, not just an engine-internal one: it's the lingua franca that lets
GPU and CPU tools interoperate.

## Composable data systems (where Sirius fits)

Modern analytical engines increasingly aren't built monolithically — they're **composed**
from reusable, swappable components:

- **front end** (parse + optimize → a logical plan): DuckDB, Spark, DataFusion
- **plan IR** (a portable plan format): Substrait
- **execution engine** (run the plan over columnar data): **libcudf** (GPU), Velox (CPU),
  DataFusion

**Sirius is a composition:** it reuses **DuckDB** as the SQL front end (parser, optimizer,
catalog, transactions — [`duckdb-extension-api.md`](duckdb-extension-api.md)) and **libcudf +
RMM** as the GPU execution engine, gluing them with its own pipeline scheduler and
**cuCascade** memory tiering. That's *why* Sirius is "thin": much of a database is delegated
to mature components, and Sirius supplies the GPU execution wiring between them.

### Cousins (other GPU SQL engines)

- **Spark RAPIDS** — Spark front end → libcudf execution (distributed).
- **HeavyDB** (formerly OmniSci/MapD) — standalone GPU analytics database.
- **Theseus** (Voltron Data) — large-scale GPU query engine.
- **BlazingSQL** — early SQL-on-cuDF engine (now defunct; team → Voltron Data).

Sirius's niche among these: **DuckDB-embedded, single-node, multi-GPU** — bring GPU speed to
the DuckDB you already use, transparently.

## Why this matters for a Sirius contributor

- When you call `cudf::…`, you're using a **shared RAPIDS API** — its semantics, idioms, and
  constraints (and the best docs/issues for them) come from **upstream RAPIDS**, not Sirius.
- Sirius being "thin" is a *design choice*, not an omission: it orchestrates RAPIDS + DuckDB
  rather than rebuilding a database. Bugs/limits often trace to a component (cuDF's int128
  gap, DuckDB plan shapes) rather than to Sirius itself.
- For GPU-side questions, the **RAPIDS / libcudf docs and GitHub** are the authoritative
  source (the `/module-context` docs are extracted from libcudf headers — see
  [`cudf.md`](cudf.md)).

## See also

- [`cudf.md`](cudf.md) / [`rmm.md`](rmm.md) — the two RAPIDS libraries Sirius uses;
  [`cucascade.md`](cucascade.md) — the complementary NVIDIA tiered-memory library.
- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — the columnar/Arrow foundation;
  [`duckdb-extension-api.md`](duckdb-extension-api.md) — the DuckDB front end Sirius composes with.
