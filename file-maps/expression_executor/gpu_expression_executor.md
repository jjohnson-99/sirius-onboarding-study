# `expression_executor/gpu_expression_executor.cpp` → Subsystem Map

Companion for Week 3, **Days 1–2** of [`onboarding-path.md`](../../onboarding-path.md)
("Expressions on GPU"), tagged effectively **read closely** — this is the heart of how
FILTER and PROJECTION turn DuckDB expression trees into GPU work. Read it with
`docs/super-sirius/expression-executor.md` open (the two map onto each other almost
section-for-section), and keep the cuDF `ast` and `unary_binary` module docs handy
(`.claude/skills/module-discover/docs/cudf/modules/{ast,unary_binary}.md`, or
`/module-context` in a Claude Code session).

Background concept first: read
[`../../reference/explainers/filters-and-expression-evaluation.md`](../../reference/explainers/filters-and-expression-evaluation.md)
before this map — it explains *what* an expression tree is and why FILTER/PROJECTION are
"just expression evaluation," which this file then implements on the GPU.

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> links to other study notes are relative to this file. Line numbers were accurate as of
> 2026-06-15 — re-grep the function name if the file has moved.

## Where this sits

This is the first **leaf-compute** subsystem in the study — not part of the
query-lifecycle spine (`sirius_interface` → `sirius_engine` → scheduler), but the thing
an *operator* reaches for when it actually needs to evaluate an expression on a batch.
Two operators are its whole reason to exist:

- **FILTER** (`src/op/sirius_physical_filter.cpp:49`) constructs one and calls
  `select(table_view)`.
- **PROJECTION** (`src/op/sirius_physical_projection.cpp:52`) constructs one and calls
  `execute(table_view)`.

(A handful of other call sites reuse it: table scan / parquet scan filter pushdown, and
the nested-loop join's per-row lambda. Same class, narrower entry points.)

It depends *downward* on **cuDF** (the AST types `cudf::ast::tree` / `compute_column`,
and the materialize-mode `cudf::unary_operation` / `binary_operation` / `cast`) and on
**RMM** (the stream + memory resource it threads into every cuDF call). It is a sibling,
not a subclass, of the operator hierarchy in
[`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md).

> **The header is the real spec.** Unusually, the most informative file here is the
> *header* `src/include/expression_executor/gpu_expression_executor.hpp` — it carries
> long Doxygen comments on the strategy, the "tree of AST trees," and `execute_result`.
> The `.cpp` is the driver; the per-expression-type logic is fanned out into
> `specializations/gpu_execute_*.cpp` (see below). Read header → driver `.cpp` →
> one or two specializations.

## The key idea first: a "tree of AST trees"

cuDF can evaluate a whole expression tree in **one fused kernel** via
`cudf::ast::tree` + `cudf::compute_column` — *if* every node has a cuDF AST equivalent.
But many SQL constructs don't (`CASE`, `LIKE`, `SUBSTRING`, `COALESCE`, unsupported
`CAST` types). Those are **AST breakers**.

So the executor walks the DuckDB expression and greedily builds AST subtrees up to each
breaker. At a breaker it **materializes** that subtree into a `cudf::column` (a
one-off kernel), stashes the column, and references it from the enclosing AST subtree via
a `cudf::ast::column_reference`. The result is a *tree of AST trees* whose edges are
breakers — each AST tree run by one `compute_column`, each breaker run by its own kernel.

```
       AND                  ← one cudf::ast::tree (compute_column)
      /   \
   a > 5   CASE …           ← breaker: materialized to a temp column,
             │                referenced from the AND tree as a column_reference
          (its own subtree, possibly more AST trees underneath)
```

This is the single most important framing; everything else is mechanism around it.

## Three strategies + the size gate

Selected by the `strategy` ctor arg (default from the
`expression_executor_strategy` SET variable; see
[`gpu_expression_translator.md`](gpu_expression_translator.md) only for the *separate*
translator — not this). Enum in
`src/include/expression_executor/expression_executor_strategy.hpp`:

| Strategy | How | cuDF API |
|---|---|---|
| `MATERIALIZE` | every node = one kernel; intermediates are real columns | `cudf::unary_operation`, `binary_operation`, `cast`, … |
| `AST_INTERPRET` *(default)* | build a `cudf::ast::tree`, one interpreting kernel | `cudf::compute_column` |
| `AST_JIT` | build the tree, JIT-compile a fused kernel | `cudf::compute_column_jit` |

**`min_ast_size`** (ctor, default `2`): a 1-operator AST gains nothing over a direct
kernel but still pays `compute_column` launch overhead. Before adding a subtree, the
executor calls `count_ast_ops()`; if the count is `< min_ast_size`, that subtree is run
operator-by-operator in MATERIALIZE mode instead. Mental model from the doc:
`MATERIALIZE` ≡ `AST_INTERPRET` with `min_ast_size = ∞`.

`execution_mode::{AST, MATERIALIZE}` is the *internal per-node* hint (not the strategy).
It is **only a hint** — a node tagged AST that turns out to be a breaker materializes
anyway, then wraps itself via `materialize_as_ast_column()` so the parent still sees an
AST reference.

## The driver `.cpp` — functions that matter

`src/expression_executor/gpu_expression_executor.cpp`:

| Function (signature) | Line | Role |
|---|---|---|
| `std::unique_ptr<cudf::table> execute(cudf::table_view input)` | 280 | **PROJECTION entry.** Resets AST state, sets `_input_table`, evaluates each expression in `MATERIALIZE` mode at the top level, post-processes (type-cast each result to the expression's `return_type`), returns one column per expression. |
| `std::unique_ptr<cudf::table> select(cudf::table_view input)` | 356 | **FILTER entry.** Asserts a single BOOLEAN expression, calls `execute(input)` to get a boolean mask column, then `cudf::apply_boolean_mask` (375) to keep passing rows. |
| `execute_result execute(duckdb::Expression const&, execution_mode)` | 378 | The **dispatcher** — a `switch` on `GetExpressionClass()` to the per-class private overloads (whose bodies live in `specializations/`). |
| `std::unique_ptr<cudf::column> execute_ast(expr_ref root)` | 215 | Builds the combined `table_view` (input columns **+** temp columns) and runs `compute_column` (237) or `compute_column_jit` (239) on the AST root. |
| `execute_result materialize_as_ast_column(col)` | 243 | The breaker bridge: stash a materialized column in `_temp_columns`, emit a `cudf::ast::column_reference` to it. |
| `void release_temporaries(...)` | 253 / 264 | Free the temp scalars/columns once their AST tree has been evaluated. |
| `std::size_t count_ast_ops(duckdb::Expression const&)` | 408 | Recursively counts the cuDF AST ops a subtree would lower to — drives the `min_ast_size` gate. Encodes the real cuDF limits (CASE→0, COALESCE→0, decimal-result function→0, unsupported CAST→0). |

### Post-processing (inside `execute`, lambda at 299)
Worth reading closely once: after a node produces a column/scalar, the executor compares
its cuDF type to the DuckDB `return_type` and inserts a `cudf::cast` (322) when they
differ — e.g. `extract(year …)` comes back `int16` but DuckDB wants `int64`. `BOUND_REF`
passes through untyped; non-fixed-width mismatches throw (the GPU can't cast
STRING/LIST/STRUCT). A small, concrete instance of the type-reconciliation theme from
[`../../reference/explainers/types-duckdb-cudf-sirius.md`](../../reference/explainers/types-duckdb-cudf-sirius.md).

## The per-type specializations

The dispatcher's `switch` lands in private `execute(BoundXExpression const&, mode)`
overloads, but their **bodies are split out**, one file per expression class, in
`src/expression_executor/specializations/`:

| File | Handles | Notable |
|---|---|---|
| `gpu_execute_reference.cpp` | `BOUND_REF` (`column #n`) | leaf; just a column view / `column_reference` |
| `gpu_execute_constant.cpp` | `BOUND_CONSTANT` (`42`, `'hi'`) | leaf; literal → scalar / AST literal |
| `gpu_execute_comparison.cpp` | `=`, `<`, `>`, `IS NOT DISTINCT FROM` | maps to cuDF AST comparison ops |
| `gpu_execute_conjunction.cpp` | `AND`, `OR` | n-ary → chained AST ops |
| `gpu_execute_between.cpp` | `x BETWEEN lo AND hi` | lowered to `(x>=lo) AND (x<=hi)` |
| `gpu_execute_cast.cpp` | `CAST(x AS T)` | AST only for fixed-width `T` (see `supported_ast_cast_types`); else breaker |
| `gpu_execute_function.cpp` | `BoundFunctionExpression` (`+ - * /`, `UPPER`, `YEAR`, …) | AST only for the 6 arithmetic `function_id`s; rest materialize |
| `gpu_execute_operator.cpp` | `BoundOperatorExpression` (`NOT`, `IS NULL`, `IN`, `COALESCE`) | biggest (602 lines); COALESCE via iterative `cudf::replace_nulls`; `IN` over the full numeric set |
| `gpu_execute_case.cpp` | `CASE WHEN …` | **always a breaker** — materializes |

The allow-lists that decide "AST or breaker" live in
`src/include/expression_executor/ast_supported_types.hpp`:
`supported_ast_cast_types` (UBIGINT, BIGINT, DOUBLE) and `supported_ast_functions`
(add, sub, mul, div, int_div, mod). Anything outside → materialize.

## The Sirius-AST migration — now complete (updated for pull `d9172de6`, 2026-06-15)

> **This section described an in-progress dual surface; that migration has since landed**
> (`#701/#870`, `#703/#880`; full diff in
> [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md)).
> The earlier study text below is kept only as history.

**As of this pull, the executor is AST-native.** The `sirius::expression` PIMPL wrapper was
**retired** (`#703`) and `src/include/expression/expression.hpp` deleted. Concretely:
- ctors take **`sirius::ast::node`** (or a `duckdb::vector<unique_ptr<node>>`); the single
  member is **`_ast_expressions`** — the old `_expressions` (DuckDB) member is **gone**.
- dispatch is `execute(sirius::ast::node const&, mode)` (header ~347) via `std::visit`, into
  **`sirius::ast::*` per-alternative overloads only** (header ~392–407) — including a **new
  `sirius::ast::aggregate` overload**. The 9 `duckdb::BoundXExpression` overloads are gone.
- it **still round-trips through `sirius::ast::to_duckdb`** internally — but only to recover
  `expression_class` + `return_type` for result **post-processing** (and the BOOLEAN
  assertion in `select`), not to evaluate. So `to_duckdb` is now a post-processing detail,
  not a "fall back to the DuckDB path" bridge.

Everything else in this map (the tree-of-AST-trees, the three strategies, `min_ast_size`,
the breakers, the `specializations/gpu_execute_*.cpp` fan-out — all 9 files still present) is
unchanged. The takeaway: **there is one input surface now (Sirius AST)**; you no longer have
to track a dual DuckDB/AST path.

<details><summary>Historical (pre-pull): the dual surface</summary>

Previously there were two parallel worlds — a DuckDB-typed path (`_expressions`,
`duckdb::BoundXExpression` overloads, `sirius::expression` PIMPL) and a newer
`sirius::ast::node` path that round-tripped back to it — an in-progress migration tracked by
[sirius-db/sirius#699](https://github.com/sirius-db/sirius/issues/699), replaced one
specialization at a time. That is the state our Week-3 reading originally described.
</details>

## Types fundamental to *this* file

- **`gpu_expression_executor::execute_result`** *(this file; header 154)* — a
  `std::variant` over `{ ast_result, cudf::column_view, unique_ptr<cudf::scalar>,
  unique_ptr<cudf::column> }`. Every per-node `execute` returns one. **Think:** "the
  node produced *either* an AST reference (deferred) *or* a concrete materialized value."
- **`ast_result`** *(this file; header 94)* — an AST node reference **plus** the indices
  of temp scalars/columns that must stay alive until that tree is evaluated. **Think:**
  "an AST pointer and its keep-alive list."
- **`cudf::ast::tree` / `cudf::ast::expression` / `column_reference`** *(cuDF)* — the
  arena that owns AST nodes, the node references, and the way to point an AST node at a
  table column (index N). **Think:** "the GPU-evaluable expression graph + how breakers
  re-enter it."
- **`cudf::compute_column` / `compute_column_jit`** *(cuDF)* — run an AST tree over a
  table, one column out, in one (interpreted or JIT-compiled) kernel. **Think:** "the
  fused expression kernel."
- **`expression_executor_strategy` / `execution_mode`** *(this file)* — the global mode
  vs. the internal per-node hint. **Think:** "what the user picked vs. what this node
  ended up doing."
- `rmm::cuda_stream_view`, `rmm::device_async_resource_ref` — the stream + allocator
  threaded into every cuDF call; see
  [`../../reference/explainers/cuda-streams-and-async.md`](../../reference/explainers/cuda-streams-and-async.md).
- The generic `duckdb::Expression` subclasses (`BoundComparisonExpression`, etc.) are
  catalogued in `docs/super-sirius/expression-executor.md`'s "Supported Expression
  Types" table — refer there rather than re-listing.

## Takeaway

FILTER/PROJECTION are thin: they hand a `table_view` and a list of expressions to this
class and get a `table` back. The cleverness is the **tree-of-AST-trees** that fuses as
much as cuDF allows into single kernels and falls back to per-node materialization at
breakers — the same "do it on the GPU in bulk, fall back where you must" instinct you
saw in the aggregate widening. Next in Week 3: a medium operator
([`../op/sirius_physical_top_n.md`](../op/sirius_physical_top_n.md) or
[`../op/sirius_physical_partition.md`](../op/sirius_physical_partition.md)) and its test,
then your first PR. The standalone translator that some joins use is
[`gpu_expression_translator.md`](gpu_expression_translator.md).
