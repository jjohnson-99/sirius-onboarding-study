# `planner/sirius_plan_comparison_join.cpp` → Plan-Builder Map (comparison join)

Companion for **Week 2 / Weeks 4–5** of [`onboarding-path.md`](../../onboarding-path.md) —
the largest plan builder (448 lines), split out of the
[plan-builders family map](plan-builders.md). Tagged **read closely**. Builds
`LOGICAL_COMPARISON_JOIN` / `DELIM_JOIN` into a Sirius hash or nested-loop join; pair with
[`../op/sirius_physical_hash_join.md`](../op/sirius_physical_hash_join.md) and the
[translator](../expression_executor/gpu_expression_translator.md).

> Source/doc paths (`src/...`) relative to the Sirius repo root; study-note links relative
> to this file. Lines as of 2026-06-16 (pull `d9172de6`) — re-grep.

## The key idea first: choose hash vs. NLJ, and pre-decide build-side uniqueness

`create_plan(LogicalComparisonJoin&)` (435) dispatches by node type: `COMPARISON_JOIN` →
`plan_comparison_join` (265), `DELIM_JOIN` → `plan_delim_join` (its own builder), `ASOF_JOIN`
→ `NotImplementedException`. The substance is `plan_comparison_join` (265), which:

1. estimates child cardinalities;
2. **proves build-side key uniqueness *before* `create_plan`** (273) — important ordering:
   `create_plan` *moves* data out of the logical nodes, so the uniqueness analysis must run
   first;
3. builds both child subtrees;
4. picks the join strategy and constructs the operator.

## Strategy selection (the branch table)

| Condition | Builds | Line |
|---|---|---|
| no conditions (cross product) | throws `NotImplementedException` → CPU | 280 |
| conditions supported by hash join **and** not `prefer_range_joins` | `sirius_physical_hash_join` | 319 |
| else, DuckDB NLJ supports it | `sirius_physical_nested_loop_join` | 411 |
| else | throws `NotImplementedException("Blockwise NLJ…")` → CPU | 425 |

`are_conditions_supported` (313, the hash-join static gate — needs ≥1 equality condition)
decides hash vs. NLJ. **Range/inequality strategies are all stubbed out:** `can_merge` /
`can_iejoin` are computed (285–302) but the piecewise-merge and IE-join branches are
**commented-out `NotImplementedException`s** (388–410) — Sirius does range joins only via the
NLJ fallback today. Honest boundary worth noting.

Conditions are turned into `sirius::join_condition` once via `wrap_join_conditions` (311);
the hash-join ctor (319) receives them plus **`op.filter_pushdown`** (329) — the dynamic
join-filter pushdown hook (see [Dynamic filters / bloom](#dynamic-filter-pushdown-hook) below).

## The interesting bit: `prove_unique_columns` → `unique_build_keys` (42, 335–374)

This builder carries a ~220-line **static uniqueness analysis** that the runtime join uses to
pick its fast path. `prove_unique_columns(LogicalOperator&)` (42) recursively returns the set
of output columns proven to form a unique key:

- **`LOGICAL_GET`** (45) — the table's **PRIMARY KEY** columns (PK only, not plain UNIQUE:
  PK guarantees NOT NULL, and nullable UNIQUE can have duplicate NULLs that break
  `distinct_hash_join`'s contract), remapped to output positions.
- **`AGGREGATE`** (93) — the group columns (single grouping set) are unique.
- **`LIMIT`/`TOP_N`** (104) — pass through child uniqueness (row truncation only).
- **`FILTER`/`ORDER_BY`/`PROJECTION`** (111/126/140) — pass through, remapped through the
  projection map (only direct `BOUND_REF` pass-throughs are safe).
- **`COMPARISON_JOIN`** (160) — per-join-type reasoning about whether each side's row-level
  uniqueness *survives* the join (e.g. INNER: left rows stay unique iff the right keys are
  unique; FULL OUTER → none, NULL-padding can duplicate).

Then in `plan_comparison_join` (346–374): if all conditions are pure equality, extract the
build-side (right) key columns, and if the **proven-unique set ⊆ build keys**, set
`hj.unique_build_keys = true`. That flag is what makes the operator pick the
**`cudf::distinct_hash_join`** fast path over `cudf::hash_join` (see the BUILD_PROBE/STANDARD
branches in [`../op/sirius_physical_hash_join.md`](../op/sirius_physical_hash_join.md)). So
this plan-time analysis is the *source* of the "unique keys" fast-path you read about in the
operator.

## Dynamic-filter pushdown hook

`op.filter_pushdown` (a `duckdb::JoinFilterPushdownInfo`) is threaded into the hash-join ctor
(329) and stored as `filter_pushdown` on the operator. This is the seam through which a join
build can push a runtime filter into a downstream scan/probe — the **dynamic-filter**
framework. As of this tree that framework is **scaffolding** (no producers/consumers wired);
see [`../../reference/explainers/bloom-filters.md`](../../reference/explainers/bloom-filters.md)
and [`../../reference/explainers/filters-and-expression-evaluation.md`](../../reference/explainers/filters-and-expression-evaluation.md)
for where bloom filters would slot in. The plumbing exists here; the behavior doesn't yet.

## Types fundamental to *this* file

- **`duckdb::LogicalComparisonJoin`** *(DuckDB)* — the logical join: `conditions`,
  `join_type`, projection maps, `filter_pushdown`, `join_stats`. **Think:** "the JOIN,
  pre-translation."
- **`sirius::join_condition` / `wrap_join_conditions`** *(Sirius)* — the Sirius-native join
  clause (holds a `sirius::ast::node` post-PIMPL-retirement); see
  [`../expression/ast.md`](../expression/ast.md). **Think:** "one ON-clause comparison."
- **`prove_unique_columns` result** *(file-local)* — proven-unique output column set, the
  input to `unique_build_keys`. **Think:** "can the build side have duplicate keys?"
- **`sirius_physical_hash_join` / `sirius_physical_nested_loop_join`** *(Sirius ops)* — the
  two products. **Think:** "equality → hash; inequality-only → NLJ."

## Takeaway

The comparison-join builder is the join *strategist*: hash join when there's an equality
condition (else NLJ, else CPU fallback; range/IE joins stubbed out), wrapping conditions into
Sirius AST and threading the dynamic-filter pushdown hook. Its standout is the **plan-time
`prove_unique_columns` analysis** that sets `unique_build_keys` → the operator's
`distinct_hash_join` fast path. Back to the family: [`plan-builders.md`](plan-builders.md);
the sibling heavyweight is [`sirius_plan_aggregate.md`](sirius_plan_aggregate.md).
