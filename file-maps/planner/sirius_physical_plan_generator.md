# `planner/sirius_physical_plan_generator.cpp` → Plan-Generation Map

Companion for Week 2, Days 1–2 of [`onboarding-path.md`](../../onboarding-path.md),
tagged **read**. Read `src/planner/sirius_physical_plan_generator.cpp` (+
`src/include/planner/sirius_physical_plan_generator.hpp`) with
`docs/super-sirius/physical-plan-generation.md` (**Part 1**) open — that doc's
operator-mapping table is the companion to this file.

> Source/doc paths are relative to the Sirius repo root (`../../sirius/` from here).
> Line numbers accurate as of 2026-06-10; re-grep if the file moved.

## Where this sits

This is the **translator**: DuckDB's optimized *logical* plan in →
Sirius *physical* plan tree out. It runs just before the engine:

```
GPUExecutionBind / OnFinalizePrepare  →  create_plan()  →  sirius_engine.initialize()
   (sirius_extension.md / transparent)    (THIS FILE)        (sirius_engine.md)
```

`sirius_extension.cpp::SiriusGeneratePhysicalPlan` (and the transparent path's
`SiriusContext::OnFinalizePrepare`) call into here. The tree it returns is wrapped in
`sirius_prepared_statement_data` and later consumed by `sirius_engine` — so this file
is the bridge between the two you've already read.

## The key idea

It's a **big type switch**. DuckDB hands a tree of `LogicalOperator` nodes; this file
walks it and, for each node `type`, dispatches to a per-operator `create_plan()`
overload that builds the matching `sirius_physical_operator`. The crucial design fact:

> **This file is the single source of truth for "does Sirius support this query?"**
> Any logical operator without a handler throws `NotImplementedException`, which
> propagates up and triggers **CPU fallback** (silent in the transparent path; the
> `run_internal_cpu_fallback_query` you saw in `sirius_extension.md`).

## Functions

| Function (signature) | Line | Role |
|---|---|---|
| `unique_ptr<op::sirius_physical_operator> create_plan(unique_ptr<duckdb::LogicalOperator> op)` | 85 | **Entry point.** Three profiled phases: `ResolveOperatorTypes()`, `ColumnBindingResolver` (resolve column references), then `create_plan(*op)` to build the tree, then `plan->verify()`. |
| `unique_ptr<op::sirius_physical_operator> create_plan(duckdb::LogicalOperator& op)` | 110 | **The dispatch switch.** `switch (op.type)` → either a supported `create_plan(op.Cast<LogicalXxx>())` overload, or `throw NotImplementedException(...)`. Copies `estimated_cardinality` onto the result. |
| `OrderPreservationType order_preservation_recursive(op&)` | 38 | Walks the tree to decide whether output order must be preserved (FIXED / NO_ORDER / INSERTION_ORDER). |
| `bool preserve_insertion_order(...)` | 59, 79 | Combines the above with the `PreserveInsertionOrderSetting` to decide the final answer. Feeds the engine's order handling. |

## The mapping (what the switch dispatches to)

The per-operator `create_plan(LogicalXxx&)` overloads are **not in this file** — they
live in sibling `src/planner/sirius_plan_*.cpp` files (one per operator). This file
only routes. The pairing (from the switch + the plan-gen doc table):

| DuckDB logical | → Sirius physical | Builder file |
|---|---|---|
| `LOGICAL_GET` | `TABLE_SCAN` | `sirius_plan_get.cpp` |
| `LOGICAL_PROJECTION` | `PROJECTION` | `sirius_plan_projection.cpp` |
| `LOGICAL_FILTER` | `FILTER` | `sirius_plan_filter.cpp` |
| `LOGICAL_AGGREGATE_AND_GROUP_BY` | `HASH_GROUP_BY` / `UNGROUPED_AGGREGATE` | `sirius_plan_aggregate.cpp` |
| `LOGICAL_COMPARISON_JOIN` / `LOGICAL_DELIM_JOIN` | `HASH_JOIN` / `NESTED_LOOP_JOIN` / delim | `sirius_plan_comparison_join.cpp` |
| `LOGICAL_ORDER_BY` | `ORDER_BY` | `sirius_plan_order.cpp` |
| `LOGICAL_TOP_N` | `TOP_N` | `sirius_plan_top_n.cpp` |
| `LOGICAL_LIMIT` | `STREAMING_LIMIT` | `sirius_plan_limit.cpp` |
| `LOGICAL_MATERIALIZED_CTE` / `LOGICAL_CTE_REF` | `CTE` / `CTE_SCAN` | `sirius_plan_cte.cpp` etc. |

**Explicitly unsupported → fallback** (each is a `throw NotImplementedException` arm
in the switch): `WINDOW`, `UNNEST`, `SAMPLE`, `ANY_JOIN`, `ASOF_JOIN`,
`CROSS_PRODUCT`, `POSITIONAL_JOIN`, set ops (`UNION`/`EXCEPT`/`INTERSECT`),
`RECURSIVE_CTE`, all DDL/DML (`INSERT`, `UPDATE`, `CREATE_*`, …). Skim the switch once
to internalize the supported/unsupported boundary — it *is* Sirius's feature surface.

## For the target query

`SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY l_returnflag` produces a
logical tree roughly `AGGREGATE_AND_GROUP_BY → GET(lineitem)`. The switch routes:
`LOGICAL_GET` → `TABLE_SCAN` (later refined to `DUCKDB_SCAN` by the engine, since
lineitem is a DuckDB table — `seq_scan`), and `LOGICAL_AGGREGATE_AND_GROUP_BY` →
`HASH_GROUP_BY` (it has a GROUP BY). The Days 4–5 reading uses the *ungrouped* sibling
as the simpler representative — same builder file, same two-phase shape.

## Types fundamental to *this* file

- **`duckdb::LogicalOperator`** — the input node type (see
  [`duckdb-types-glossary.md`](../../reference/duckdb-types-glossary.md)). The whole
  file is a walk over this tree. **Think:** the *what-to-compute* tree.
- **`op::sirius_physical_operator`** *(Sirius base)* — the output node type. **Think:**
  the *how-on-GPU* tree (detail in
  [`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md)).
- **`duckdb::LogicalOperatorType`** — the enum the switch keys on; learning which
  values have arms here = learning Sirius's supported feature set. **Think:** the
  catalog of SQL constructs, each either handled or thrown.
- **`ColumnBindingResolver`** *(DuckDB)* — rewrites column references into resolved
  bindings before translation. **Think:** "make every column reference concrete first."

## Takeaway

This file decides *what* the GPU plan is (and whether the GPU can run the query at
all); [`../sirius_engine.md`](../sirius_engine.md) decides *how* it runs (pipelines).
The per-operator `sirius_plan_*.cpp` builders are out of scope for Week 2 — the
mapping table above is enough; you'll open individual builders only when adding or
fixing a specific operator.
