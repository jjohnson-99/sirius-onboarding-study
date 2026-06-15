# Week 3 Concepts — The Operator & Expression Layer (and your first PR)

The Week 3 checkpoint in [`onboarding-path.md`](../onboarding-path.md):

> First contribution submitted. Understand operator + expression layer.

Week 2 ([`week2-concepts.md`](week2-concepts.md)) traced one query **across** the engine —
the horizontal spine from SQL string to result. Week 3 turns **downward**: into what a
single operator actually *does* on the GPU, and especially how the most common operators
(FILTER, PROJECTION) evaluate arbitrary SQL expressions. Then you ship something small.

This is a shift in shape: there's no single new end-to-end trace. The through-line is
**"how does an operator compute a column?"** — answered once for expressions (Days 1–2)
and once for a self-contained operator (Day 3), then exercised by making a change
(Days 4–5).

## Part 1 — Expressions on GPU (Days 1–2)

**File maps:**
[`expression_executor/gpu_expression_executor.md`](../file-maps/expression_executor/gpu_expression_executor.md),
[`expression_executor/gpu_expression_translator.md`](../file-maps/expression_executor/gpu_expression_translator.md)
· **doc:** `docs/super-sirius/expression-executor.md` · **prime with:**
[`../reference/explainers/filters-and-expression-evaluation.md`](../reference/explainers/filters-and-expression-evaluation.md)

Recall from Week 2 that FILTER and PROJECTION were operators in the plan. What makes them
work is a shared subsystem, `gpu_expression_executor`:

- **PROJECTION** calls `execute(table_view)` → a new table, one column per output
  expression.
- **FILTER** calls `select(table_view)` → evaluate a boolean expression to a mask, then
  `cudf::apply_boolean_mask`.

The central idea — and the thing to actually internalize this week — is the **"tree of
AST trees."** cuDF can evaluate an expression tree in one fused kernel
(`cudf::compute_column` over a `cudf::ast::tree`), but only for nodes that have a cuDF AST
equivalent. Constructs that don't (`CASE`, `LIKE`, `SUBSTRING`, `COALESCE`, unsupported
`CAST`) are **AST breakers**: the executor materializes them into a temp column with a
one-off kernel and references that column from the surrounding AST tree. The result is a
tree whose internal trees are GPU-fused and whose edges are breakers.

Two dials control this: the **strategy** (`MATERIALIZE` / `AST_INTERPRET` /`AST_JIT`, a
SET variable) and **`min_ast_size`** (don't bother building an AST for a 1-op subtree —
the kernel-launch overhead isn't worth it). A clean mental model from the doc:
`MATERIALIZE` is just `AST_INTERPRET` with `min_ast_size = ∞`.

Two things that look confusing until named:
1. **The per-type logic is fanned out** into `specializations/gpu_execute_*.cpp` (one
   file per expression class). The driver `.cpp` is a dispatcher; the specializations are
   where comparison/cast/function/CASE/IN each decide "emit AST node vs. materialize."
2. **The input surface is the Sirius AST** (`sirius::ast::node`). *(Updated for pull
   `d9172de6`: this used to be a dual DuckDB/AST surface mid-migration — that migration has
   since landed (`#701/#703`), the `sirius::expression` PIMPL is retired, and the executor is
   now AST-native. It still round-trips through `sirius::ast::to_duckdb` internally, but only
   to recover types for result post-processing — not to evaluate. See the
   [executor map](../file-maps/expression_executor/gpu_expression_executor.md) and the
   [post-pull report](../reports/post-pull-review-2026-06-15.md).)*

The **translator** (`gpu_expression_translator`) is a separate, smaller utility: it only
*builds* a cuDF AST and hands it off (no evaluation, no breakers — `nullopt` if it can't),
for mixed/inequality joins that pass the AST to `cudf::mixed_join`. Know the
**evaluate-here vs. translate-and-hand-off** distinction; you'll meet the translator for
real in Weeks 4–5.

> Type reconciliation recurs here: the executor casts each result to the expression's
> DuckDB `return_type` (e.g. `extract(year …)` int16 → int64), and refuses non-fixed-width
> casts — a concrete instance of
> [`../reference/explainers/types-duckdb-cudf-sirius.md`](../reference/explainers/types-duckdb-cudf-sirius.md).

## Part 2 — A medium operator (Day 3)

**File maps:**
[`op/sirius_physical_top_n.md`](../file-maps/op/sirius_physical_top_n.md) *(start here)*,
[`op/sirius_physical_partition.md`](../file-maps/op/sirius_physical_partition.md)
*(harder alternative)* · **prime with:**
[`../reference/explainers/sorting-and-order-by.md`](../reference/explainers/sorting-and-order-by.md)

Day 3 picks one operator and reads it *with its test*. The two choices teach different
lessons:

- **TOP-N** (`ORDER BY … LIMIT n`) is the gentle one and the recommended start. It reuses
  the **two-phase local→merge** shape from Week 2's aggregate: each batch keeps its top
  `offset+limit` rows, the merge concatenates and takes the global top. Its real interest
  is **per-case cuDF primitive selection** in `compute_top_n_table`: `top_k_order` +
  `gather` + `sort_by_key` for a non-null single key, but a **full-sort fallback** for a
  nullable key (because `top_k_order` can't honor `NULLS FIRST/LAST`). That "the fast
  primitive can't express the SQL semantics, so degrade to the correct one" move — plus
  applying `offset` only at the merge — is the lesson.

- **PARTITION** is the on-ramp to the hard part of the codebase. It's a source **and**
  sink that hashes key columns to route rows into `N` partitions so joins and grouped
  aggregates can run in parallel (and across GPUs). Its interest isn't a kernel — it's
  **control**: deriving partition keys from the *parent* operator, sizing `N` from input
  bytes with a multi-GPU floor, and coordinating a build/probe sibling pair under careful
  double-locking. That's a preview of Weeks 4–5 (scheduling, task hints, ports).

Either way, the habit to form is **read the operator alongside `test/cpp/operator/`** —
the tests are the fastest way to see an operator run in isolation, and they're where your
first contribution will likely live.

## Part 3 — Ship something small (Days 4–5)

No new file-map — this is workflow. The natural first PRs from what you've now read:

- add an SQL logic test under `test/sql/`, or a C++ operator test under
  `test/cpp/operator/` (you've just read two operators' tests);
- extend a supported case — e.g. an expression type in a `gpu_execute_*` specialization,
  or an ordering-expression case in TOP-N (currently `BOUND_REF` only);
- fix a small issue.

Process: read `CONTRIBUTING.md`, run `pre-commit run -a` (clang-format / clang-tidy /
codespell — see [`../reference/explainers/testing-sirius.md`](../reference/explainers/testing-sirius.md)),
open the PR. The weekend reading is `docs/super-sirius/configuration.md` (the SET knobs —
including `expression_executor_strategy`, which you just learned controls Part 1).

## Where this leaves you

You can now answer "how does an operator turn a batch into a batch" — for the
expression-driven operators (the executor's tree-of-AST-trees) and for a self-contained
compute operator (TOP-N's primitive selection), and you've seen the *edge* of the
concurrency machinery (PARTITION). That's the operator + expression layer the checkpoint
asks for. **Weeks 4–5** go after the genuinely hard core that PARTITION hinted at:
pipeline execution, the task scheduler, the scan subsystem, and the hash join — where the
bugs are races and leaks, not wrong kernels.
