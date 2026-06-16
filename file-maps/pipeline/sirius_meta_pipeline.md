# `pipeline/sirius_meta_pipeline.cpp` → Pipeline-Construction Map (Step 4a)

Companion for **Week 4** of [`onboarding-path.md`](../../onboarding-path.md), the
**Step-4 / pipeline-construction** reading, tagged **read**. This is **Step 4a** —
the first half of pipeline construction, read *before*
[`sirius_pipeline_converter.md`](sirius_pipeline_converter.md) (Step 4b–4d). Pair with
`docs/super-sirius/execution-flow.md` Step 4a.

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines as of 2026-06-16 (pull `d9172de6`) — re-grep if moved.

## The key idea first: group pipelines by their shared sink

A **meta-pipeline** is "a set of pipelines that all have the same sink." Step 4a builds a
*tree* of these from the physical-operator plan, so that blocking operators (joins,
group-bys, sorts) become sinks with their own child meta-pipelines. This is **DuckDB's
meta-pipeline pattern, ported to Sirius** — the header comments restate DuckDB's build
rules almost verbatim:
1. for joins, build the **blocking (build) side before the probe side** — the streaming
   pipeline depends on it;
2. build **child pipelines last** (e.g. a hash join becomes a *source* for FULL OUTER scan
   of its hash table only after the probe is built).

So Step 4a produces a dependency-ordered pipeline tree; Step 4b
([the converter](sirius_pipeline_converter.md)) then rewrites it into Sirius execution
pipelines. Mental model: **4a = "carve the plan into pipelines by sink"; 4b = "split each
pipeline into Sirius's GPU shapes."**

## Where this sits

Driven directly by the engine ([`../sirius_engine.md`](../sirius_engine.md),
`initialize_internal`):

```
root = make_shared<sirius_meta_pipeline>(build_ctx, state, /*sink=*/nullptr)   sirius_engine.cpp:371
root->build(*plan)        :102   recursively carve pipelines     · Step 4a
root->ready()             :110   reverse operator lists, mark ready
root->get_pipelines(...)         hand the tree to the converter  ─▶ sirius_pipeline_converter.md
```

The output (a `sirius_meta_pipeline` tree) is exactly the converter's input.

## How `build()` carves pipelines (102) — `build_sirius_pipelines` recursion

`build()` (102) seeds the base pipeline and calls `build_sirius_pipelines(node, current)`,
which recurses the operator tree and, per operator's own `build_pipelines()` override
(in `src/op/`, the base in [`../op/sirius_physical_operator.md`](../op/sirius_physical_operator.md)):
- **streaming op** (FILTER, PROJECTION) → appended to the *current* pipeline as an
  intermediate operator;
- **blocking op** (HASH_JOIN, ORDER_BY, GROUP_BY) → becomes a **sink**; its build input
  gets a **child meta-pipeline** (`create_child_meta_pipeline`, 120) built first;
- **source op** (scan) → set as the pipeline's source.

`ready()` (110) then **reverses** each pipeline's operator list (built sink-first, run
source-first) and marks them ready — after which `source`/`sink` are just aliases for the
first/last operators (the invariant the executor relies on).

## Methods worth knowing

| Method (signature) | Line | Role |
|---|---|---|
| `void build(op& op)` | 102 | entry: carve the plan into this meta-pipeline's pipelines (excl. the shared sink). |
| `void ready()` | 110 | reverse operator lists + mark all pipelines ready (recursive). |
| `void build_sirius_pipelines(op& node, sirius_pipeline& current)` | (hdr) | the recursion that dispatches to each operator's `build_pipelines()`. |
| `sirius_meta_pipeline& create_child_meta_pipeline(current, op)` | 120 | new child meta-pipeline `current` depends on — the build-side-first rule. |
| `sirius_pipeline& create_pipeline()` | 134 | empty pipeline within this meta-pipeline. |
| `void create_child_pipeline(current, op, last)` | 254 | child pipeline of `current` (e.g. hash-join-as-source). |
| `void get_pipelines(result, recursive)` | 47 | flatten the tree for the converter / progress tracking. |

## Types fundamental to *this* file

- **`sirius_meta_pipeline`** *(this file)* — a group of pipelines sharing one sink, with a
  parent + children forming the dependency tree. **Think:** "one sink and everything that
  feeds it."
- **`MetaPipelineType` `{REGULAR, JOIN_BUILD}`** *(this file)* — whether the shared sink is
  a join build (drives the build-before-probe ordering). **Think:** "is this the build side
  of a join?"
- **`sirius_pipeline`** *(`sirius_pipeline.hpp`)* — the individual pipeline (ordered
  operators, source/sink); see [`sirius_pipeline_converter.md`](sirius_pipeline_converter.md).
  **Think:** "one source→sink chain."
- **`sirius_pipeline_build_state` / `pipeline_build_context`** *(plan-time)* — bookkeeping
  (batch indices, dependencies) + config threaded through the build. **Think:** "the build's
  scratchpad."

## Takeaway

Step 4a is the DuckDB-style "carve the plan into pipelines grouped by sink, build-side
first" pass — small (274 lines) and mostly tree bookkeeping, with the real per-operator
decisions delegated to each operator's `build_pipelines()`. Its tree feeds directly into
the heavy lifting next door: [`sirius_pipeline_converter.md`](sirius_pipeline_converter.md)
(Step 4b–4d), where the Sirius-specific splitting happens.
