# Filters & Expression Evaluation

Once you have batches of columns ([`vectorized-execution.md`](vectorized-execution.md)),
the next question is: how does an operator actually *compute* something over them —
`l_extendedprice * (1 - l_discount)` for a projection, or `l_shipdate >= DATE '1994-01-01'
AND l_quantity < 24` for a filter? That's **expression evaluation**, and it's the
per-operator computation primitive underneath FILTER, PROJECTION, and every join
condition and aggregate argument.

> **Prime around:** Week 3 — the expression executor is the Week 3 Days 1–2 read
> (`docs/super-sirius/expression-executor.md`, `src/expression_executor/`); load the cuDF
> `ast` / `unary_binary` module docs (`.claude/skills/module-discover/docs/cudf/modules/`;
> or via `/module-context` in a Claude Code session).

## An expression is a tree

A SQL scalar expression is an **AST** (abstract syntax tree): leaves are **column
references** or **constants**; internal nodes are operators (`+`, `*`, `>`, `AND`) or
functions. `l_extendedprice * (1 - l_discount)`:

```
        *
       / \
extprice   -
          / \
         1   discount
```

The engine receives these already **bound** — DuckDB resolves each leaf to a concrete
column index (a `BoundReferenceExpression`), so the evaluator knows "leaf = column 3 of
the input batch." (You saw exactly this in
[`aggregation-and-group-by.md`](aggregation-and-group-by.md): an aggregate's argument is a
bound reference.)

Expressions show up in four places: **projections** (compute output columns), **filters**
(a boolean predicate), **join conditions**, and **aggregate arguments**. One evaluator
serves them all.

## Two operators, one primitive

- **Projection** — evaluate expression(s) → produce new output column(s). Sirius:
  `gpu_expression_executor::execute(batch)`.
- **Filter (selection)** — evaluate a **boolean** predicate → a mask, then keep the rows
  where it's true. Sirius: `gpu_expression_executor::select(batch)`.

A filter is thus two steps: **evaluate → compact**. The predicate produces a boolean
column (`[T,F,T,T,F,…]`); then **stream compaction** keeps only the true rows, producing a
smaller *contiguous* result (`cudf::apply_boolean_mask`). On a GPU compaction is itself a
parallel **prefix-sum (scan)** — compute each surviving row's output position, then scatter
— so even "just filtering" is a real parallel algorithm.

## How to evaluate over a batch: three strategies

This is the part that matters for performance, and it's a spectrum from simple to fast:

| Strategy | How | Cost |
|---|---|---|
| **Materialize** | walk the tree; each node produces a **full intermediate column** (`1 - discount` → temp col; `extprice * temp` → result) | simplest & most robust, but every intermediate is a full column **written then re-read** — lots of memory traffic |
| **AST interpret** | hand the whole tree to a vectorized interpreter that evaluates it in ~one pass (cuDF's `ast::compute_column`) | far fewer intermediates; one kernel for the expression |
| **JIT / fuse** | compile the tree to a single fused kernel; intermediates live in **registers**, never in memory | fastest — no interpreter overhead, minimal memory traffic |

**Why the spectrum matters — and why it matters *more* on a GPU.** Every materialized
intermediate is a full column read+written to memory, and DB work is
**memory-bandwidth-bound** ([`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md)).
Fusing the expression keeps intermediates in registers and collapses N passes over memory
into one — exactly the resource a GPU is most starved for. So "how you evaluate
expressions" is a real performance lever, not a detail.

## Filter pushdown (don't evaluate what you can skip)

The cheapest predicate is one you never run. Filters on scan columns can be **pushed into
the scan**: skip whole Parquet row groups whose min/max stats can't satisfy the predicate,
and read fewer columns ([`parquet-format.md`](parquet-format.md)). Sirius pushes table
filters into the scan operator (`src/planner/sirius_plan_get.cpp`); only the residual
predicate is evaluated as a FILTER operator. Predicate *ordering* matters too — apply
cheap, selective predicates first so later ones see fewer rows.

## Sirius specifics

- **`gpu_expression_executor`** is the evaluator; it has a configurable **strategy knob**,
  `expression_executor_strategy` = `materialize` | `ast_interpret` | `ast_jit` (plus
  `use_cudf_expr`) — the three strategies above, selectable at runtime.
- A **translator** converts DuckDB's bound expression tree into Sirius's own AST
  (`sirius::ast::node`) which the executor then evaluates (interpret) or compiles (jit) —
  often via cuDF's `cudf::ast` / transform machinery.
- **FILTER** → `select` (predicate + compaction); **PROJECTION** → `execute` (new columns)
  — see `docs/super-sirius/operators.md`.
- **Footguns** (honest): special-cased paths for regex (`enable_regex_jit_impl`) and
  strings; and **BIGINT arithmetic (`+`,`-`,`*`) falls back to CPU** because the GPU lacks
  an INT128 accumulator — the same int128 limitation behind the BIGINT-`SUM` fallback in
  [`aggregation-and-group-by.md`](aggregation-and-group-by.md).

## See also

- [`vectorized-execution.md`](vectorized-execution.md) — the batches expressions run over.
- [`gpu-vs-cpu-for-databases.md`](gpu-vs-cpu-for-databases.md) — why fusing (fewer
  intermediates) is a big deal on a bandwidth-bound GPU.
- [`hash-join-build-probe.md`](hash-join-build-probe.md) — join conditions are
  expressions (the `MIXED_JOIN` mode evaluates a cuDF AST predicate); and
  [`file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md)
  for the FILTER/PROJECTION operators in code.
