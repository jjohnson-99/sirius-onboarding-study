# `expression_executor/gpu_expression_translator.cpp` → Utility Map

Companion for Week 3, **Days 1–2** of [`onboarding-path.md`](../../onboarding-path.md),
tagged **read** (supporting). Read **after**
[`gpu_expression_executor.md`](gpu_expression_executor.md) and the
`docs/super-sirius/expression-executor.md` "GPU Expression Translator" section — this is
the *separate, smaller* sibling utility, easy to confuse with the executor.

> Source/doc paths are relative to the Sirius repo root; study-note links are relative to
> this file. Lines accurate as of 2026-06-15 — re-grep if the file moved.

## Don't confuse it with the executor

| | `gpu_expression_executor` | `gpu_expression_translator` *(this file)* |
|---|---|---|
| Job | **evaluate** an expression against a batch, return columns/filtered rows | **translate** an expression into a standalone `cudf::ast::tree` and hand it back |
| Returns | `cudf::table` (real data) | `translated_expression` (a tree + owned literals) — *no evaluation* |
| Used by | FILTER, PROJECTION, scans | **mixed/inequality joins** (`sirius_physical_hash_join` MIXED_JOIN mode) |
| On unsupported input | builds a "tree of AST trees," materializing breakers | returns `std::nullopt` → caller falls back to row-by-row |

The executor *embeds* AST-building inside a larger evaluate-or-materialize machine; the
translator *only* builds the tree, for a caller (cuDF's `mixed_join`) that wants a cuDF
AST expression to apply itself. No materialization, no breakers-as-temp-columns — if it
can't translate, it bails with `nullopt`.

## Where this sits

Downstream consumer: `src/op/sirius_physical_hash_join.cpp`, MIXED_JOIN mode — it needs
inequality conditions (`a.x < b.y`) expressed as a cuDF AST so `cudf::mixed_join()` can
evaluate them on the GPU during the join. So this file is really part of the join story
(Week 4–5); you meet it in Week 3 only because it lives in `expression_executor/`.
Depends on cuDF AST (`cudf::ast::tree`, `ast_operator`) and RMM (an owned stream for the
literal scalars' lifetime).

## The result type (and a lifetime gotcha worth seeing)

```cpp
struct translated_expression {
    cudf::ast::tree tree;                                   // the built AST
    rmm::cuda_stream owned_stream;                          // declared BEFORE literals…
    std::vector<std::unique_ptr<cudf::scalar>> owned_literals;  // …so it outlives them
};
```

cuDF AST literals hold **references** to `cudf::scalar`s, so the tree is only valid while
those scalars live — hence the struct co-owns them. The header calls out that
`owned_stream` is declared *before* `owned_literals` on purpose: C++ destroys members in
**reverse** declaration order, so `cudaStreamDestroy` fires *after* the scalars are
freed. A nice, real example of GPU-async lifetime discipline (see
[`../../reference/explainers/cuda-streams-and-async.md`](../../reference/explainers/cuda-streams-and-async.md)).

## Functions that matter

`src/expression_executor/gpu_expression_translator.cpp`:

| Function (signature) | Line | Role |
|---|---|---|
| `optional<translated_expression> translate_expression(sirius::ast::node const&, …)` | 214 | Top-level: translate a whole expression; `nullopt` if anything inside is unsupported. |
| `optional<translated_expression> translate_expression_with_names(…)` | 227 | Same, but resolves column refs **by name** via a `column_name_resolver_fxn` (for parquet filter pushdown, where there's no positional cuDF table). |
| `optional<translated_expression> translate_join_condition(sirius::join_condition const&)` | 287 | One equality/inequality join condition → AST. |
| `optional<translated_expression> translate_join_conditions(conditions, start, end, swap_sides)` | 299 | Combine a range of conditions with `AND`; `swap_sides` flips LEFT/RIGHT refs for RIGHT/OUTER joins. **This is the mixed-join entry point.** |
| `optional<expr_ref> add_expression(...)` (overload set) | 356, 383, 525, 570, 610, 631, 639, 670, 698, 707, … | The recursive per-node builders (one overload per `sirius::ast` alternative), each appending nodes to `tree` and returning the new node's reference — or `nullopt`. |
| `add_join_condition(...)` | 242 | Builds `left <op> right`, translating both sides. |

Note the signatures now take **`sirius::ast::node`**, not `duckdb::Expression` — this
file was flipped to the native Sirius AST input ahead of the executor (repo commit
*"flip gpu_expression_translator to sirius::ast::node input"*; cf. the migration note in
[`gpu_expression_executor.md`](gpu_expression_executor.md)).

## What it refuses to translate (→ `nullopt` → row-by-row fallback)

Per the doc and the `nullopt` returns scattered through `add_expression`:

- `CASE` expressions
- `COALESCE`, `TRY`
- `CAST` to non-fixed-width types (e.g. VARCHAR)
- parameter expressions
- `DISTINCT` operators — `IS DISTINCT FROM` actually **throws** `NotImplementedException`

The pattern is uniform: any child returning `nullopt` propagates up, so the whole
translation is all-or-nothing (unlike the executor, which materializes the un-translatable
part and keeps going).

## Types fundamental to *this* file

- **`translated_expression`** *(this file)* — the AST tree plus the owned scalars/stream
  keeping it valid. **Think:** "a self-contained, handed-off cuDF expression."
- **`column_name_resolver_fxn`** = `std::function<std::string(duckdb::idx_t)>` *(this
  file)* — maps a bound-reference index to a column *name*, for name-based AST refs.
  **Think:** "positional ref → named ref, for parquet pushdown."
- **`sirius::join_condition`** *(Sirius; PIMPL over `duckdb::JoinCondition`)* — one
  `left <comparison> right` join clause. **Think:** "one ON-clause comparison."
- **`cudf::ast::tree` / `ast_operator`** *(cuDF)* — see
  [`gpu_expression_executor.md`](gpu_expression_executor.md); same types, here built for
  export rather than immediate evaluation.

## Takeaway

A focused converter: DuckDB/Sirius expression → cuDF AST, all-or-nothing, for callers
(mixed joins) that evaluate the AST themselves. You'll come back to it for real in
Weeks 4–5 with `sirius_physical_hash_join`. For Week 3, the point is just to see the
*two* expression paths clearly: **evaluate-here** (the executor) vs. **translate-and-hand-off**
(this). Back to the Week 3 thread:
[`../op/sirius_physical_top_n.md`](../op/sirius_physical_top_n.md).
