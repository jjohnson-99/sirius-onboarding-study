# `pipeline/sirius_meta_pipeline.cpp` → Pipeline-Construction Map (Step 4a)

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), the
**Step-4 / pipeline-construction** reading, tagged **read** (but it rewards a closer look —
this version goes deep). It's **Step 4a** — the first half of pipeline construction, read
*before* [`sirius_pipeline_converter.md`](sirius_pipeline_converter.md) (Step 4b–4d). Pair with
`docs/super-sirius/execution-flow.md` Step 4a, and keep
[`sirius_pipeline.md`](sirius_pipeline.md) open (this file *builds* those).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this file.
> Line numbers accurate as of 2026-06-17 — re-grep if moved.

## The key idea first: group pipelines by their shared sink

A **meta-pipeline** is "a set of pipelines that all have the **same sink**." Step 4a turns the
physical-operator tree into a *tree of meta-pipelines*, so each blocking operator (join,
group-by, sort) becomes a sink with its own child meta-pipeline that must finish first. This is
**DuckDB's meta-pipeline pattern, ported to Sirius** — the header (hpp 43–53) restates DuckDB's
two build rules almost verbatim:

1. **For joins, build the blocking (build) side before the probe side** — the streaming probe
   pipeline gets a *dependency* on the build (a dependency *across* meta-pipelines).
2. **Build child pipelines last** — e.g. a hash join only becomes a *source* (scan its hash
   table for a FULL OUTER join) after the probe pipeline is built.

Mental model: **4a = "carve the plan into pipelines grouped by sink, build-side first"; 4b =
"split each pipeline into Sirius's GPU shapes."** Step 4a is mostly *tree + dependency
bookkeeping*; the real per-operator decisions are delegated to each operator's
`build_pipelines()` override.

## Where this sits

Driven directly by the engine ([`../sirius_engine.md`](../sirius_engine.md),
`initialize_internal`):

```
root = make_shared<sirius_meta_pipeline>(build_ctx, state, /*sink=*/nullptr)  sirius_engine.cpp:371
  └─ ctor calls create_pipeline()        :28/134   seed the (sink-less) base pipeline
root->build(*plan)                        :102      recurse the operator tree, carving pipelines  · Step 4a
root->ready()                             :110      reverse each pipeline's operators, mark ready
root->get_pipelines(result, recursive)    :47       flatten the tree ─▶ sirius_pipeline_converter.md
```

The output — a `sirius_meta_pipeline` *tree*, flattened to a list of `sirius_pipeline`s — is
exactly the converter's input.

## The data model: two nested structures (read this first)

The confusing part of this file is that "the tree" is really **two** parent/child relationships,
and they mean different things. Hold them apart:

| Structure | Field | What it groups | Dependency meaning |
|---|---|---|---|
| **pipelines within one meta-pipeline** | `pipelines` (hpp 141) | pipelines that share **one sink** but have **different sources** (e.g. UNION ALL legs; a hash-join-as-source child) | ordered by the **`dependencies` map** (hpp 143) — *intra*-meta-pipeline |
| **child meta-pipelines** | `children` (hpp 146) | a **different** sink (the build side of a join, a CTE, the query's RESULT_COLLECTOR feeding into it) | the child's base pipeline is a `dependency` of the parent pipeline — *cross*-meta-pipeline |

So `children` = "other sinks I depend on" (build-before-probe); `pipelines` + `dependencies` =
"several source→sink chains into the *same* sink, in what order." `get_base_pipeline()` (cpp 42)
is just `pipelines[0]`, the first chain into this sink.

## The recursion in detail — driven by operators, not this file

`build()` (cpp 102) does almost nothing itself: it calls **`op.build_pipelines(base_pipeline,
*this)`** (cpp 107) and lets the *operator* drive. The base implementation lives in
[`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md)
(`sirius_physical_operator.cpp:145`) and is the heart of Step 4a:

```
operator::build_pipelines(current, meta_pipeline)        operator.cpp:145
├─ is_sink()?  (HASH_JOIN build, ORDER_BY, GROUP_BY, …)               :149
│     ├─ set_pipeline_source(current, *this)   ── THIS op feeds `current` (it's current's source)
│     └─ child = meta_pipeline.create_child_meta_pipeline(current, *this)   ── new sink = this op  :157
│           └─ child.build(children[0])         ── recurse the BUILD side into the child (rule 1)
└─ not a sink:                                                         :159
      ├─ children.empty()  → set_pipeline_source(current, *this)      ── a scan: pipeline source  :163
      └─ else (FILTER/PROJECTION/streaming) →
            add_pipeline_operator(current, *this)                     ── append as intermediate    :168
            children[0]->build_pipelines(current, meta_pipeline)      ── keep walking down         :169
```

Note this builds each pipeline **sink-first / top-down** (the sink op is reached first, the scan
last), which is why `ready()` later reverses the operator list.

**Composite operators override `build_pipelines`** to add the rule-2 "as-source" child and extra
dependencies — e.g. `sirius_physical_hash_join` (`hash_join.cpp:368`),
`sirius_physical_nested_loop_join` (`:235`), `sirius_physical_delim_join`, `sirius_physical_cte`,
and `sirius_physical_result_collector` (`:87`, which wraps the whole plan in a child
meta-pipeline). They call back into *this* file's helpers (`create_child_meta_pipeline`,
`create_child_pipeline`, `add_recursive_dependencies`).

## The build helpers (what the operators call back into)

- **`create_child_meta_pipeline(current, op)` (cpp 120)** — rule 1. Push a new child
  meta-pipeline whose **sink is `op`** (the build operator), set its `parent = &current`, and
  **`current.add_dependency(child->get_base_pipeline())`** (cpp 128) — the cross-meta-pipeline
  "build must finish before probe" edge. Propagates `recursive_cte` to the child.
- **`create_pipeline()` (cpp 134)** — append an empty `sirius_pipeline` to `pipelines`, set its
  sink (this meta-pipeline's sink) and a fresh batch index via
  `state.set_pipeline_sink(..., next_batch_index++)`.
- **`create_child_pipeline(current, op, last_pipeline)` (cpp 254)** — rule 2, the "as-source"
  child (hash-join scanning its HT for FULL OUTER). Requires `current` fully built
  (`current.source` set), forks a child via `state.create_child_pipeline` (see
  [`sirius_pipeline.md`](sirius_pipeline.md)), inherits `current.base_batch_index`, and makes the
  child depend on `current` **plus** every pipeline added since `last_pipeline`
  (`add_dependencies_from`, cpp 141, false = exclusive of `last_pipeline`).
- **`add_dependencies_from(dependent, start, including)` (cpp 141)** — the intra-meta-pipeline
  dependency helper: make `dependent` depend on all pipelines created after (optionally
  including) `start`.

## `ready()` and batch indexes

`ready()` (cpp 110) walks every pipeline calling `sirius_pipeline::is_ready()` (which
**reverses** the operator list — built sink-first, run source-first) and recurses into children.
After this, `source`/`sink` are reliable aliases for the first/last operators — the invariant the
[task](gpu_pipeline_task.md)/[executor](gpu_pipeline_executor.md) rely on.

Each pipeline gets a `base_batch_index = next_batch_index++ * BATCH_INCREMENT` (cpp 99/137), the
ordering key the pipeline uses for its active-batch multiset (see
[`sirius_pipeline.md`](sirius_pipeline.md) "Batch indexes"). The huge `BATCH_INCREMENT` gives each
pipeline a non-overlapping numeric range.

## Honesty notes — a lot here is DuckDB-inherited scaffolding

This file carries more DuckDB shape than Sirius actually exercises. Don't over-invest:

- **`build_sirius_pipelines(node, current)` (declared hpp 106) and `Type()` (declared hpp 84) are
  never defined anywhere**, and the `type` member (hpp 137) is never initialized or read.
  *(The previous version of this map wrongly called `build_sirius_pipelines` "the recursion that
  dispatches to each operator's `build_pipelines()`" — it's a dead declaration; the real driver
  is `operator::build_pipelines` above.)*
- **Unused outside this file** (DuckDB features not wired into Sirius's planner): the
  **finish-event** machinery (`add_finish_event` cpp 205, `has_finish_event` 220,
  `get_finish_group` 225, for double-Finalize ops like IEJoin — and IE/range joins aren't
  implemented in Sirius, see [comparison-join builder](../planner/sirius_plan_comparison_join.md)),
  **`create_union_pipeline`** (cpp 232, UNION ALL same-sink legs), **`set_recursive_cte`** /
  **`assign_next_batch_index`** — none are called from anywhere but this file.
- **`add_recursive_dependencies` (cpp 166)** *is* called (by hash_join / NLJ after building the
  child pipeline), but its actual dependency-adding loop is **commented out** (cpp 189–202); it
  early-returns on `recursive_cte` and otherwise just traverses — effectively a no-op today.

## Methods

| Method (signature) | Line | Role |
|---|---|---|
| `sirius_meta_pipeline(ctx, state, sink)` | cpp 22 | construct with a shared `sink` (nullptr for the root); seeds the base pipeline via `create_pipeline()`. |
| `void build(op&)` | cpp 102 | entry: `op.build_pipelines(base_pipeline, *this)` — hand control to the operator recursion. |
| `void ready()` | cpp 110 | reverse each pipeline's operators (`is_ready`) + recurse children. |
| `create_child_meta_pipeline(current, op)` | cpp 120 | **rule 1**: new child meta-pipeline (sink = `op`); `current` depends on its base pipeline. |
| `void create_child_pipeline(current, op, last)` | cpp 254 | **rule 2**: the as-source child (e.g. hash-join FULL-OUTER scan), with its intra-meta deps. |
| `sirius_pipeline& create_pipeline()` | cpp 134 | append an empty pipeline (sink + batch index set). |
| `void add_dependencies_from(dep, start, incl)` | cpp 141 | intra-meta-pipeline dependency helper. |
| `void get_pipelines(result, recursive)` | cpp 47 | flatten pipelines (this + children) — the converter's input. |
| `void get_meta_pipelines(result, rec, skip)` | cpp 58 | flatten the meta-pipeline tree. |
| `get_base_pipeline()` / `get_last_child()` / `get_dependencies(p)` | cpp 42 / 71 / 82 | `pipelines[0]` / deepest child / intra-meta deps of a pipeline. |
| `assign_next_batch_index(p)` | cpp 97 | batch index = `n++ * BATCH_INCREMENT` *(unused externally)*. |
| `create_union_pipeline(current, order)` | cpp 232 | UNION-ALL same-sink leg *(unused externally)*. |
| `add_finish_event` / `has_finish_event` / `get_finish_group` | cpp 205 / 220 / 225 | per-pipeline finish events *(unused externally; IEJoin-style)*. |
| `add_recursive_dependencies(deps, last_child)` | cpp 166 | called by joins, but body commented out — near no-op. |
| *(declared, undefined)* `build_sirius_pipelines` / `Type` | hpp 106 / 84 | **vestigial** — see honesty notes. |

## Member fields (hpp)

| Field | Line | Meaning |
|---|---|---|
| `sink` | 135 | the operator all `pipelines` here drain into (nullptr at root). |
| `pipelines` | 141 | same-sink, different-source chains (base + union/child pipelines). |
| `dependencies` | 143 | **intra**-meta-pipeline ordering between those pipelines. |
| `children` | 146 | **child meta-pipelines** (other sinks this one depends on — build-before-probe). |
| `parent` | 133 | the pipeline (in the parent meta-pipeline) that this one feeds. |
| `state` / `build_ctx` | 131 / 129 | the shared `sirius_pipeline_build_state` + plan-time config. |
| `next_batch_index` | 148 | counter for assigning pipeline batch-index ranges. |
| `type` / `recursive_cte` | 137 / 139 | `type` unused; `recursive_cte` propagated but its dep logic is commented. |
| `finish_pipelines` / `finish_map` | 151 / 153 | finish-event bookkeeping *(unused)*. |

## Types fundamental to *this* file

- **`sirius_meta_pipeline`** *(this file)* — a group of pipelines sharing one sink, with a parent
  + child meta-pipelines forming the dependency tree. **Think:** "one sink and everything that
  feeds it."
- **`MetaPipelineType {REGULAR, JOIN_BUILD}`** *(hpp 33)* — declared to mark whether the shared
  sink is a join build, but `Type()` is undefined and the field unused. **Think:** "intended
  build-vs-regular tag — currently inert."
- **`sirius_pipeline`** *(mapped in [`sirius_pipeline.md`](sirius_pipeline.md))* — the individual
  source→operators→sink chain this file assembles. **Think:** "one chain; the meta-pipeline owns
  a set of them."
- **`sirius_pipeline_build_state`** *(in [`sirius_pipeline.md`](sirius_pipeline.md))* — the
  `friend` assembler this file calls (`set_pipeline_sink`, `create_child_pipeline`). **Think:**
  "the thing actually allowed to mutate a pipeline's source/operators/sink."
- **`pipeline_build_context`** *(`pipeline/pipeline_build_context.hpp`)* — plan-time config
  (`preserve_insertion_order`, …) threaded through every meta-pipeline. **Think:** "the build's
  read-only config."

## Takeaway

Step 4a carves the physical plan into a **tree of meta-pipelines** (each = one sink + its feeding
chains), enforcing DuckDB's two rules — build-side before probe (via child meta-pipelines +
cross-pipeline dependencies) and child "as-source" pipelines last. The recursion is driven by
**each operator's `build_pipelines()`**, with this file providing the tree/dependency bookkeeping
helpers; a good chunk of the DuckDB-inherited surface (finish events, unions, recursive-CTE deps,
`Type`/`build_sirius_pipelines`) is **vestigial in Sirius**. The resulting flattened pipeline list
feeds the heavy lifting next door — [`sirius_pipeline_converter.md`](sirius_pipeline_converter.md)
(Step 4b–4d), where the Sirius-specific operator splitting happens.
