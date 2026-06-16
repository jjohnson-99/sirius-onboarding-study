# `planner/sirius_plan_aggregate.cpp` → Plan-Builder Map (aggregate)

Companion for **Week 2/3** of [`onboarding-path.md`](../../onboarding-path.md) — the
heavyweight plan builder split out of the [plan-builders family map](plan-builders.md).
Tagged **read**. Builds the `LOGICAL_AGGREGATE_AND_GROUP_BY` node into a Sirius
grouped/ungrouped aggregate; pair with the operator maps
([grouped](../op/sirius_physical_grouped_aggregate.md) /
[ungrouped](../op/sirius_physical_ungrouped_aggregate.md)) and the
[AST entry](../expression/ast.md).

> Source/doc paths (`src/...`) relative to the Sirius repo root; study-note links relative
> to this file. Lines as of 2026-06-16 (pull `d9172de6`) — re-grep.

## The key idea first: pick grouped vs. ungrouped, hoist exprs, downcast HUGEINT

`create_plan(LogicalAggregate&)` (316) is dispatched from the
[generator](sirius_physical_plan_generator.md) switch. It does four things, in order:

1. **Downcast HUGEINT → BIGINT** (`downcast_hugeint_types`, 303) — DuckDB widens
   `sum(int32)` to `HUGEINT` (int128) to avoid overflow, but **cuDF has no int128**, so the
   plan rewrites those return types to BIGINT up front (so every downstream operator + the
   result collector agree). The same int128 footgun you saw in the ungrouped-aggregate
   operator, handled here at the *plan* level.
2. **Hoist aggregate inputs into a projection** (`extract_aggregate_expressions`, 63) — pull
   each aggregate's child/filter sub-expressions (and the group expressions) out into a
   `push_projection` fed *upstream* of the aggregate, replacing them with `BOUND_REF`s. So
   the aggregate operator only ever sees column references, not arbitrary expressions. The
   hoisted exprs are translated to Sirius AST via `translate_expressions` → `from_duckdb`.
3. **Choose simple vs. hash** — if any aggregate lacks `function.simple_update`,
   `can_use_simple_aggregation = false`.
4. **Construct the operator** (the branch table below).

## The branch table (what gets built)

| Condition | Builds | Line |
|---|---|---|
| no groups **and** simple aggregation | `sirius_physical_ungrouped_aggregate` | 351 |
| no groups, **not** simple | throws `NotImplementedException("Non simple aggregation…")` → CPU | 360 |
| has groups | `sirius_physical_grouped_aggregate` (one of three eligibility paths below) | 369 / 384 / 398 |

> **Honest note — the three grouped paths build the *same* operator (today).** The grouped
> branch first tests `can_use_partitioned_aggregate` (132) and `can_use_perfect_hash_aggregate`
> (204), but **all three arms (369 partitioned, 384 perfect-hash, 398 fallback) construct an
> identical `sirius_physical_grouped_aggregate` with identical arguments** — the computed
> `partition_columns` / `required_bits` are **not passed to the constructor**. So these are
> currently *eligibility-checking scaffolding* (ported from DuckDB's planner) that doesn't
> yet change the built operator. Worth knowing so you don't go hunting for a behavioral
> difference that isn't wired up. (The actual partitioning happens later, in the
> [converter's](../pipeline/sirius_pipeline_converter.md) `HASH_GROUP_BY → PARTITION +
> MERGE_GROUP_BY` split.)

## The two eligibility checks (read for the DuckDB-porting flavor)

- **`can_use_partitioned_aggregate`** (132) — walks child operators (through PROJECTION /
  FILTER, remapping group columns) down to the `TABLE_SCAN`, and asks the table function's
  `get_partition_info` whether the source is already **single-value-partitioned** by the
  group keys. (BOUND_REF groups only.)
- **`can_use_perfect_hash_aggregate`** (204) — small **integer** group keys whose min/max
  range (from stats, or type bounds for tiny types) fits in a bit budget
  (`PerfectHtThresholdSetting`); rejects DISTINCT / non-combinable aggregates. The
  `required_bits_for_value` (115) / `get_range_hugeint` (125) helpers support it.

Both are faithful ports of DuckDB's perfect-hash/partitioned-aggregate gating — useful as a
read on *how* DuckDB decides these, even though (per the note above) the result isn't yet
differentiated in the built operator.

## What it does **not** do

The AVG→SUM/COUNT decomposition, the two-phase local→merge split, and the actual
`cudf::groupby`/`cudf::reduce` live in the **operators**, not here
([grouped](../op/sirius_physical_grouped_aggregate.md) /
[ungrouped](../op/sirius_physical_ungrouped_aggregate.md)), and the local→merge *pair* is
minted by the [converter](../pipeline/sirius_pipeline_converter.md). This builder only
chooses grouped-vs-ungrouped, hoists/translates expressions, and fixes the HUGEINT type.

## Types fundamental to *this* file

- **`duckdb::LogicalAggregate`** *(DuckDB)* — the logical node: `groups`, `expressions`
  (the aggregates), `grouping_sets`, `group_stats`. **Think:** "the GROUP BY, pre-translation."
- **`translate_expressions(...)`** *(file-local, 47)* — bulk `from_duckdb` over a drained
  expr vector; `nullptr` slot per unsupported expr (fallback). **Think:** "the AST producer
  call for aggregates/groups." See [`../expression/ast.md`](../expression/ast.md).
- **`push_projection`** *(`sirius_plan_projection_utils.hpp`)* — inserts the upstream
  projection that the hoist produces. **Think:** "make the aggregate see only column refs."
- **`sirius_physical_{grouped,ungrouped}_aggregate`** *(Sirius ops)* — the products. **Think:**
  "logical aggregate → physical aggregate operator."

## Takeaway

The aggregate builder is the GROUP BY translator: downcast HUGEINT (cuDF int128 footgun),
hoist aggregate inputs into an upstream projection, translate expressions to Sirius AST, and
pick grouped vs. ungrouped. Its perfect-hash/partitioned eligibility checks are ported from
DuckDB but **not yet wired to change the operator** — a good honest example to remember. Back
to the family: [`plan-builders.md`](plan-builders.md); the sibling heavyweight is
[`sirius_plan_comparison_join.md`](sirius_plan_comparison_join.md).
