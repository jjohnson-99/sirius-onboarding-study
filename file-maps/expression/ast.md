# `src/expression/` (Sirius AST) → Expression-Entry Map

Companion for **Week 3** of [`onboarding-path.md`](../../onboarding-path.md) ("Expressions
on GPU"), tagged **read** — the **producer entry** into the expression subsystem. Read this
*before* the consumer maps
([`../expression_executor/gpu_expression_executor.md`](../expression_executor/gpu_expression_executor.md)
/ [`gpu_expression_translator.md`](../expression_executor/gpu_expression_translator.md)):
it's where a DuckDB expression *becomes* the `sirius::ast::node` those consume.

> Source/doc paths (`src/...`) are relative to the **Sirius repo root**; study-note links
> are relative to this file. Line numbers as of 2026-06-16 (pull `d9172de6`) — re-grep.

> **One map for the `src/expression/` subtree.** This covers the AST itself
> (`src/expression/ast/` + `node.cpp`/`utils.cpp`) and the two translators that bracket it
> (`from_duckdb.cpp`, `to_duckdb.cpp`), plus the small leaf helpers (`value.cpp`,
> `join_condition.cpp`, `function_id.cpp`, `aggregate_id.cpp`). ~2,350 lines total, but
> mostly small per-alternative headers.

## Why this exists (and why you couldn't find "the entry")

After the **PIMPL retirement** (`#701/#703`; see the executor map's migration note), the
whole expression layer is built on **`sirius::ast::node`** — a Sirius-native expression AST,
*not* `duckdb::Expression`. So there are two halves, and only the *consumer* half was mapped
before:

```
PLAN TIME (producer — THIS map):
  plan builders  src/planner/sirius_plan_{filter,projection,aggregate,top_n,...}.cpp
    └─ sirius::ast::from_duckdb(duckdb_expr)   src/expression/from_duckdb.cpp:249   ← THE entry
         → std::unique_ptr<sirius::ast::node>   stored on the operator (e.g. filter's `expression`)

EXECUTE TIME (consumer — executor/translator maps):
  operator  e.g. sirius_physical_filter::execute (filter.cpp:48)
    └─ gpu_expression_executor(*expression, …)  evaluates the node tree on the GPU
         └─ (internally) sirius::ast::to_duckdb(node)   src/expression/to_duckdb.cpp:265
              recovers DuckDB type info for result post-processing only
```

So **`from_duckdb` is the literal entry into the expression subsystem**: DuckDB bound
expression → Sirius AST, done once at plan time by the plan builders.

## The AST: `sirius::ast::node` (`src/include/expression/ast/node.hpp`)

`node` is a **struct wrapping a `std::variant`** over **12 alternatives** (node.hpp:97–109):

```
reference, constant,                              ← leaves
comparison, conjunction, between, case_expr,
cast, unary_op, coalesce, in_list,               ← interior
function_call, aggregate
```

Each alternative is a small struct in its own header (`ast/reference.hpp`, `ast/cast.hpp`,
…), storing children as `std::unique_ptr<node>`. Things worth knowing:
- **Move-only** (node.hpp:130–133) — trees own their children; no copying.
- **Alternative order is ABI** (node.hpp:92–95): `std::visit` dispatch indexes by position;
  new alternatives go *at the end*.
- **Compile-time contract** (node.hpp:63–114): every alternative must implement
  `cudf_ast_op_count()` — enforced by a `concept` + `static_assert` so a missing method is a
  one-line named error, not a deep template blowup. (The *bodies* live in
  [`ast_op_counter.cpp`, mapped with the executor](../expression_executor/gpu_expression_executor.md);
  `node::cudf_ast_op_count()` at node.cpp:71 just `std::visit`s into them.)
- **Typed accessors** (`holds<T>`, `get<T>`, `as_reference/as_aggregate/...`,
  `require_reference/require_aggregate` for operator boundaries — node.cpp:47–94).

Maps to the same 12 per-alternative `execute(sirius::ast::X const&)` overloads in the
[executor](../expression_executor/gpu_expression_executor.md) — that's the 1:1 the variant
exists to enable.

## The entry: `from_duckdb()` (`src/expression/from_duckdb.cpp`)

| Function (signature) | Line | Role |
|---|---|---|
| `std::unique_ptr<node> from_duckdb(duckdb::Expression const&)` | 249 | **The public entry.** `switch` on `GetExpressionClass()` → per-class `translate_*` helper; recurses into children. |
| `translate_reference` / `translate_constant` | 84 / 90 | leaves: index + type / `Value`→`sirius` payload (via `sirius::from_duckdb(value, type)`). |
| `translate_comparison` / `translate_conjunction` / `translate_between` / `translate_case` / … | 97 / 123 / 136 / 151 | interior nodes; each maps the DuckDB subtype to a Sirius alternative. |
| `translate_children(...)` | 71 | recurse a child vector; **`nullopt` if any child is unsupported**. |

**The fallback contract (important):** `from_duckdb` returns **`nullptr`** when it meets an
expression subtype the GPU can't lower (the `default: return nullptr` arms). Callers (plan
builders) treat null as "fall back" — store a null slot, and the operator/query degrades to
CPU. It *throws* only on a genuinely unknown `ExpressionClass`, or when an embedded
`Value`/`LogicalType` is unsupported (`not_implemented_exception` from the `sirius::from_duckdb`
leaf helpers). This is the **first** place a query can be rejected for GPU — upstream of every
`NotImplementedException` you saw in the operator/executor maps (cf.
[`../fallback.md`](../fallback.md)).

## The reverse: `to_duckdb()` (`src/expression/to_duckdb.cpp`)

`to_duckdb(node const&)` (265) `std::visit`s each alternative back to a `duckdb::Expression`.
It is **not** part of evaluation — the executor uses it only to recover `expression_class` +
`return_type` for result post-processing (and the round-trip-equivalence property:
`from_duckdb → to_duckdb → executor` ≡ executing the original). Don't read it as a hot path.

## The leaf helpers (small; read only if a type surprises you)

- `value.cpp` (160) — `sirius::from_duckdb(duckdb::Value, type)`: literal → Sirius value
  (the constant payload). Throws on unsupported value types.
- `join_condition.cpp` — `sirius::comparison_type` + `from_duckdb(duckdb::ExpressionType)`
  (the comparison-operator mapping shared by comparisons and join conditions; see the
  [translator](../expression_executor/gpu_expression_translator.md) + [hash join](../op/sirius_physical_hash_join.md)).
- `function_id.cpp` / `aggregate_id.cpp` — string function/aggregate name → Sirius enum
  (the allow-list keys the executor's AST-eligibility checks use).

## Where it's entered (the producer call sites)

`sirius::ast::from_duckdb` / `wrap` is called from the **plan builders** when building each
operator (so the operator stores its AST ready for execute-time):

`src/planner/sirius_plan_filter.cpp` (FILTER predicate), `sirius_plan_projection.cpp`
(SELECT list), `sirius_plan_aggregate.cpp`, `sirius_plan_top_n.cpp`, `sirius_plan_get.cpp`,
`sirius_plan_delim_join.cpp`, … plus a few operators that translate at runtime
(`sirius_physical_table_scan.cpp`, the scan `*_gpu_ingestible.cpp`,
`sirius_physical_result_collector.cpp`). *(The per-node plan builders are mapped in
[`../planner/plan-builders.md`](../planner/plan-builders.md) — they're the producer call
sites, one `from_duckdb` per node.)*

## Types fundamental to *this* file

- **`sirius::ast::node`** *(this subtree)* — the move-only `std::variant` over 12 expression
  kinds; the currency of the whole expression layer. **Think:** "Sirius's own
  `duckdb::Expression`, GPU-lowerable."
- **the 12 alternatives** *(`ast/*.hpp`)* — one struct per expression kind, children as
  `unique_ptr<node>`. **Think:** "the variant's cases = the executor's `execute(X)`
  overloads."
- **`from_duckdb` / `to_duckdb`** *(this subtree)* — DuckDB→Sirius (the entry, `nullptr` =
  fallback) and Sirius→DuckDB (post-processing only). **Think:** "the two bridges bracketing
  the AST."
- **`comparison_type` / `function_id` / `aggregate_id`** *(leaf enums)* — Sirius-native
  operator/function/aggregate identifiers. **Think:** "the dictionaries `from_duckdb` maps
  names into."

## Takeaway

`src/expression/` is the **producer entry** the consumer maps were missing: plan builders
call `from_duckdb` (the literal entry, with `nullptr`=fallback) to turn DuckDB expressions
into the move-only `sirius::ast::node` variant, which operators then hand to the
[executor](../expression_executor/gpu_expression_executor.md) (FILTER/PROJECTION/scan/NLJ)
or [translator](../expression_executor/gpu_expression_translator.md) (mixed joins). With
this mapped, the expression layer reads end-to-end: **build the AST here → evaluate it
there.**
