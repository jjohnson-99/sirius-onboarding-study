# `fallback` → CPU-Fallback Map (with a path correction)

Companion for **Week 6** of [`onboarding-path.md`](../onboarding-path.md), tagged **read** —
how Sirius gracefully falls back to DuckDB CPU. Read alongside
`docs/super-sirius/dynamic-filters.md` and `docs/super-sirius/optimizations.md`.

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — re-grep.

> ⚠️ **Path correction — read first.** The plan (and `CLAUDE.md`) point at
> **`src/fallback.cpp`**, but that path **does not exist**. The only file matching is
> **`src/legacy/fallback.cpp`** — and it's **legacy `gpu_processing` code**: a
> `FallbackChecker` over `GPUPhysicalOperator` / `GPUPhysicalProjection` in `namespace
> duckdb`, compiled out by default (the dead path `CLAUDE.md` itself tells you to ignore).
> **It is *not* the Super Sirius fallback.** The live fallback isn't one file — it's a
> *mechanism* spread across the entry layer and the operators. This map describes the real
> one. (Worth raising with the team: the doc/plan reference is stale.)

## The key idea first: fallback is "if the GPU can't, replay on DuckDB CPU"

Sirius supports a subset of SQL on the GPU. When a query (or part of it) isn't supported,
or GPU execution errors at runtime, Sirius **re-runs the original SQL on DuckDB's normal
CPU engine** so the user still gets a correct answer. There are two complementary
flavors:

1. **Whole-query fallback** (the live "fallback.cpp" equivalent): catch a plan/execution
   failure and replay the original query on CPU.
2. **Pre-execution rejection** (per operator/expression): translation code that hits an
   unsupported case throws `NotImplementedException`, which surfaces *before* or *during*
   GPU execution and routes into flavor 1.

## Flavor 1 — whole-query replay (the actual live code)

Lives in **`src/sirius_extension.cpp`**, already mapped in
[`sirius_extension.md`](sirius_extension.md):

- **`run_internal_cpu_fallback_query(context, conn, query, ...)`** (`sirius_extension.cpp:103`
  as of pull `d9172de6`; was 100)
  — constructs a DuckDB connection and runs the original SQL on the CPU engine.
- It's invoked from **`GPUExecutionFunction`** on two paths (see the
  [`sirius_extension.md` call sequence](sirius_extension.md#call-sequence)):
  - `if (plan_error)` → fall back (the plan couldn't be built for the GPU at all);
  - `catch` around `iface->sirius_execute_query(...)` → fall back (GPU execution threw).
- The transparent doorway has its analogue (the optimizer hooks simply don't swap in the
  Sirius physical plan when the plan can't be built, so DuckDB runs it on CPU).

> **The S3 caveat (a real footgun).** There is **no CPU fallback for `s3://` reads**:
> Sirius reads S3 only on the GPU path, so if GPU execution of an S3 query fails,
> `run_internal_cpu_fallback_query` raises a clear *"S3 CPU fallback is not supported"*
> error instead of replaying (`sirius_extension.cpp:105–114`). For local/non-S3 reads it
> replays the pre-rewrite original query. Know this before assuming "fallback always
> saves you."

## Flavor 2 — where the GPU path *rejects* a query (the throws you've already seen)

These are the conditions that *trigger* flavor 1. You've met most of them in earlier maps —
this is the consolidated list, the practical answer to "when does Sirius fall back?":

| Source of rejection | Example | Map |
|---|---|---|
| Plan generator: unsupported operator/feature | window functions, ASOF join | [`planner/sirius_physical_plan_generator.md`](planner/sirius_physical_plan_generator.md) |
| Aggregates: DISTINCT / multi-arg | `SUM(DISTINCT x)` | [`op/sirius_physical_ungrouped_aggregate.md`](op/sirius_physical_ungrouped_aggregate.md) |
| Expression executor: non-fixed-width cast, etc. | `CAST(x AS VARCHAR)` mismatch | [`expression_executor/gpu_expression_executor.md`](expression_executor/gpu_expression_executor.md) |
| Translator: CASE/COALESCE/DISTINCT in mixed join | `IS DISTINCT FROM` in ON | [`expression_executor/gpu_expression_translator.md`](expression_executor/gpu_expression_translator.md) |
| TOP-N: non-`BOUND_REF` ordering key | `ORDER BY a+b LIMIT k` | [`op/sirius_physical_top_n.md`](op/sirius_physical_top_n.md) |
| Type footguns | BIGINT `SUM` (no GPU int128 accumulator) | [`op/sirius_physical_ungrouped_aggregate.md`](op/sirius_physical_ungrouped_aggregate.md) |

The common mechanism is `duckdb::NotImplementedException` (and friends) thrown from
translation/execute, caught at the extension boundary → flavor 1 replay. This is also why
"add a fallback case" or "extend a supported case" is a clean first/non-trivial
contribution (Week 7): you're moving a case from flavor 2 (reject→CPU) to GPU-native.

## The two companion docs (Week 6 reading)

- **`docs/super-sirius/dynamic-filters.md`** — runtime-computed predicates (a hash-join
  build pushing a filter into a downstream scan/probe, etc.). **Honesty note:** the doc
  says the current code is **framework scaffolding only — no producers, no consumers, no
  behavior change yet**; concrete use cases (dynamic table-filter pushdown) land in later
  phases. So read it as *design intent*, not shipping behavior. It relates to fallback only
  loosely (both are about "do less work / degrade gracefully").
- **`docs/super-sirius/optimizations.md`** — a *catalog* of shipped perf optimizations
  (adaptive partition count, 4-phase sort, adaptive join BUILD_PROBE, drain-and-restart,
  …) with PR refs and code paths. Use it as an index: most entries point at files you've
  already mapped (partition, sort, hash join, task creator). Not fallback per se — it's the
  "what else has been tuned" tour the plan pairs with this week.

## Types / terms fundamental here

- **`run_internal_cpu_fallback_query`** *(Sirius; `sirius_extension.cpp`)* — the live
  whole-query CPU replay. **Think:** "GPU couldn't; re-run the SQL on DuckDB."
- **`duckdb::NotImplementedException`** *(DuckDB)* — the signal an operator/translator emits
  for an unsupported case; caught → fallback. **Think:** "the GPU path's polite 'not me.'"
- **`sirius_dynamic_filter` / `sirius_dynamic_filter_set`** *(Sirius; scaffolding)* — the
  dynamic-filter framework base + channel. **Think:** "wiring for runtime predicates —
  present, not yet active."
- **legacy `FallbackChecker`** *(`src/legacy/fallback.cpp`, dead)* — the old
  `gpu_processing` static plan checker. **Think:** "the file the plan *names* but you should
  *not* read for Super Sirius."

## Takeaway

Sirius's CPU fallback is a *mechanism*, not the file the plan names: unsupported features
throw `NotImplementedException` (flavor 2), and the extension layer catches plan/execution
failures and replays the original SQL on DuckDB CPU (flavor 1, `run_internal_cpu_fallback_query`
in [`sirius_extension.md`](sirius_extension.md)) — except for S3, which has no fallback.
The Week-6 companion docs are forward-looking (dynamic filters: scaffolding) and a tour
(optimizations: catalog). This closes the "graceful degradation" trio with the memory maps
([`memory/sirius_memory_reservation_manager.md`](memory/sirius_memory_reservation_manager.md),
[`downgrade/downgrade_executor.md`](downgrade/downgrade_executor.md)) — synthesized in
[`../weeks/week6-7-concepts.md`](../weeks/week6-7-concepts.md).
