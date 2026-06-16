# `planner/sirius_plan_*.cpp` → Plan-Builders Map (the per-node translators)

Companion for **Week 2** of [`onboarding-path.md`](../../onboarding-path.md)
(logical→physical) and **Week 3** (expressions) — these are the **producer call sites** that
invoke `sirius::ast::from_duckdb` ([`../expression/ast.md`](../expression/ast.md)) to attach
expression ASTs to operators. Tagged **read**. Read *after* the dispatcher,
[`sirius_physical_plan_generator.md`](sirius_physical_plan_generator.md) — this map covers
the ~18 per-node builder files it dispatches to.

> Source/doc paths (`src/...`) are relative to the **Sirius repo root**; study-note links
> relative to this file. Line numbers as of 2026-06-16 (pull `d9172de6`) — re-grep.

> **One overview map for the `sirius_plan_*.cpp` family.** They're small (32–127 lines) and
> structurally identical; the two big ones (`aggregate` 412, `comparison_join` 448, plus
> `get` 265) get extra notes. Per-file maps would be 18× the same shape.

## The key idea first: one builder per DuckDB logical node

The plan generator's `create_plan(LogicalOperator&)` is a **`switch` on `op.type`**
([generator map](sirius_physical_plan_generator.md); switch at `sirius_physical_plan_generator.cpp:120`)
that casts to the concrete logical node and calls `create_plan(op.Cast<LogicalX>())` — and
**each of those overloads lives in its own `sirius_plan_X.cpp`**. So the family is the
*implementation* of the generator's switch, one file per case. Every builder does the same
three things:

1. **recurse** — `create_plan(*op.children[i])` to build child operators first;
2. **construct** the Sirius operator(s) for this node (`make_uniq<op::sirius_physical_*>`);
3. **translate expressions** — call `sirius::ast::from_duckdb(*duckdb_expr)` for any
   predicates / select lists / keys, storing the resulting `sirius::ast::node` on the
   operator. *This is the expression-subsystem entry in action* — the AST map's "producer"
   side, one call site per builder.

The output is the `sirius_physical_operator` tree that the engine then hands to Step 4
([pipeline construction](../pipeline/sirius_pipeline_converter.md)).

## The builder table

| Builder file | DuckDB logical node | Sirius operator(s) produced | `from_duckdb`? | Map |
|---|---|---|---|---|
| `sirius_plan_get.cpp` (265) | `LOGICAL_GET` | `table_scan` (+ pushed-down `filter`) | ✓ (table filters) | [scan](../op/sirius_physical_duckdb_scan.md) |
| `sirius_plan_filter.cpp` (90) | `LOGICAL_FILTER` | `filter` | ✓ (predicate) | [filter→executor](../expression_executor/gpu_expression_executor.md) |
| `sirius_plan_projection.cpp` (85) + `…_projection_utils.cpp` (182) | `LOGICAL_PROJECTION` | `projection` | ✓ (select list) | [executor](../expression_executor/gpu_expression_executor.md) |
| `sirius_plan_aggregate.cpp` (412) | `LOGICAL_AGGREGATE_AND_GROUP_BY` | `grouped_aggregate` *or* `ungrouped_aggregate` | ✓ (group keys, agg args) | **own map: [sirius_plan_aggregate](sirius_plan_aggregate.md)** |
| `sirius_plan_comparison_join.cpp` (448) | `LOGICAL_COMPARISON_JOIN` / `DELIM_JOIN` | `hash_join` *or* `nested_loop_join` | ✓ (join conditions) | **own map: [sirius_plan_comparison_join](sirius_plan_comparison_join.md)** |
| `sirius_plan_order.cpp` (50) | `LOGICAL_ORDER_BY` | `order` | ✓ (sort keys) | — |
| `sirius_plan_top_n.cpp` (43) | `LOGICAL_TOP_N` | `top_n` | ✓ (order + limit) | [top_n](../op/sirius_physical_top_n.md) |
| `sirius_plan_limit.cpp` (78) | `LOGICAL_LIMIT` | `streaming_limit` | ✓ | [limit](../op/sirius_physical_limit.md) |
| `sirius_plan_delim_join.cpp` (127) | (delim-join splitting) | `right/left_delim_join`, `grouped_aggregate` | ✓ | — |
| `sirius_plan_cte.cpp` (67) / `…_recursive_cte.cpp` (92) | `MATERIALIZED_CTE` / `CTE_REF` | `cte`, `column_data_scan` | ✓ (cte) | — |
| `sirius_plan_column_data_get.cpp` (35) | `LOGICAL_CHUNK_GET` | `column_data_scan` | – | — |
| `sirius_plan_delim_get.cpp` (38) | `LOGICAL_DELIM_GET` | `column_data_scan` | ✓ | — |
| `sirius_plan_expression_get.cpp` (67) | `LOGICAL_EXPRESSION_GET` | `column_data_scan` | ✓ | — |
| `sirius_plan_dummy_scan.cpp` (32) | `LOGICAL_DUMMY_SCAN` | `dummy_scan` | ✓ | — |
| `sirius_plan_empty_result.cpp` (32) | `LOGICAL_EMPTY_RESULT` | `empty_result` | ✓ | — |

## The heavyweight builders (own maps)

The two largest builders carry real logic and have **dedicated maps**:
- **[`sirius_plan_aggregate.md`](sirius_plan_aggregate.md)** (412 lines) — grouped-vs-ungrouped
  choice, HUGEINT→BIGINT downcast, expression hoisting, perfect-hash/partitioned eligibility
  scaffolding.
- **[`sirius_plan_comparison_join.md`](sirius_plan_comparison_join.md)** (448 lines) — hash-vs-NLJ
  strategy, the `prove_unique_columns` analysis → `unique_build_keys` (distinct-hash-join fast
  path), and the dynamic-filter-pushdown hook.

The next-largest, kept here in the family:
- **`sirius_plan_get.cpp` (265)** — the `LOGICAL_GET` scan builder: resolves the table
  function, sets up column ids / projection / **table-filter pushdown** (translating
  `TableFilterSet` via `from_duckdb`), and may insert a pushed-down `filter`. The front door
  for the whole scan subsystem.

## The `from_duckdb` connection (why this map exists)

The [AST map](../expression/ast.md) called these builders "the producer call sites" — this
*is* them. Concretely (e.g. `sirius_plan_filter.cpp`):

```
create_plan(LogicalFilter& op):
  child = create_plan(*op.children[0])
  ast   = translate_expressions(op.expressions)   // → sirius::ast::from_duckdb per expr
  return make_uniq<sirius_physical_filter>(…, std::move(ast), …)   // operator now holds the AST
```

A `nullptr` from `from_duckdb` (unsupported expression) propagates as a null AST slot → the
operator/query degrades to **CPU fallback** ([`../fallback.md`](../fallback.md)). So the
builders are the earliest point where GPU-eligibility of a query's *expressions* is decided.

## Not a builder: `query.cpp`

`src/planner/query.cpp` (80) defines the runtime **`query`** object (the pipeline vector +
an id→pipeline index map, `build_indices`), constructed in **Step 5** by
`SiriusContext::create_query` from the engine's `new_scheduled`. It lives in `planner/` but
is execution plumbing, not a plan builder — you meet it in
[`../sirius_engine.md`](../sirius_engine.md) (execute) and
[`../pipeline/task_scheduler.md`](../pipeline/task_scheduler.md), not here.

## Unsupported nodes (honest boundary)

The generator's switch has **many commented-out / throwing cases** (WINDOW, UNNEST, SAMPLE,
ASOF_JOIN, set ops, DML, …) — no builder exists for them, so they throw
`NotImplementedException` → CPU fallback. The builder table above *is* the supported-node
list; everything else falls back. (This is the operator-level companion to the
expression-level fallback above.)

## Types fundamental to *this* file family

- **`create_plan(duckdb::LogicalX&)` overloads** *(one per builder)* — the per-node
  translators. **Think:** "the generator's switch, implemented one file per case."
- **`duckdb::LogicalOperator` subtypes** (`LogicalGet`, `LogicalFilter`, `LogicalAggregate`,
  `LogicalComparisonJoin`, …) *(DuckDB)* — the optimized logical plan nodes being translated.
  **Think:** "the input; one builder consumes each."
- **`sirius::ast::node` via `from_duckdb`** — the expression AST each builder attaches; see
  [`../expression/ast.md`](../expression/ast.md). **Think:** "the builder's expression output."
- **`sirius_physical_operator`** *(Sirius)* — the builders' product; the tree the engine builds
  pipelines from. **Think:** "logical node in → physical operator out."

## Takeaway

The `sirius_plan_*.cpp` family *is* the body of the plan generator's switch: one builder per
supported DuckDB logical node, each recursing into children, constructing the Sirius
operator, and calling `from_duckdb` to attach expression ASTs (the producer entry into the
expression subsystem). With these mapped, logical→physical translation is complete
end-to-end: dispatcher ([generator](sirius_physical_plan_generator.md)) → per-node builders
(here) → expression ASTs ([AST](../expression/ast.md)) → operators → Step 4
([construction](../pipeline/sirius_pipeline_converter.md)).
