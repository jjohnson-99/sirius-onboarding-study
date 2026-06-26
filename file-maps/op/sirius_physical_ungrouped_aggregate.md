# `op/sirius_physical_ungrouped_aggregate.cpp` → Operator Map (loop-closer)

Companion for Week 2, **Days 4–5** of [`onboarding-path.md`](../../onboarding-path.md),
tagged **read** — the operator that "closes the loop" on the end-to-end trace. Read
with the `UNGROUPED_AGGREGATE` / `MERGE_AGGREGATE` entries in
`docs/super-sirius/operators.md` and the `UNGROUPED_AGGREGATE` splitting diagram in
`docs/super-sirius/physical-plan-generation.md` (Part 3). Reference the cuDF
reduce/aggregation module docs (`.claude/skills/module-discover/docs/cudf/modules/`; or load
via `/module-context` in a Claude Code session).

> Paths relative to the Sirius repo root. Lines as of 2026-06-10.

> **All line numbers below refreshed for pull `d9172de6` (2026-06-15).** This file was
> reorganized (`#926`, plus the AST-native expression changes); the two-phase local→merge
> structure is unchanged. **Behavior change (`#926`):** `make_avg_column` (231) now keeps
> **decimal AVG out of `long double`** — it divides the merged SUM by the merged COUNT
> **on-device** (`cudf::binary_operation` after casts, ~249–262) rather than copying scalars
> to the host and round-tripping through `long double`, eliminating that precision loss. See
> [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md).

## A note on the target query

The Week 2 trace query — `SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY
l_returnflag` — actually has a GROUP BY, so it uses the *grouped* aggregate
(`HASH_GROUP_BY` + `MERGE_GROUP_BY`, via `cudf::groupby`). The plan deliberately has
you read the **ungrouped** sibling instead: it's the same two-phase structure with far
less code (`cudf::reduce` over a whole column instead of grouped hashing), so you
learn the pattern cleanly. Everything here transfers to the grouped case. Think of
this as `SELECT SUM(l_quantity) FROM lineitem` (no GROUP BY) — the simpler cousin. For where
the *actual* grouping happens (`cudf::groupby`), see
[`sirius_physical_grouped_aggregate.md`](sirius_physical_grouped_aggregate.md).

## The big idea: aggregation is two-phase (local → merge)

A pipeline runs in parallel over many input batches, so you can't sum a column in one
shot. Sirius splits aggregation into two operators (the pipeline converter creates the pair
in `construct_sirius_specific_operator`, see
[`../pipeline/sirius_pipeline_converter.md`](../pipeline/sirius_pipeline_converter.md)):

1. **`sirius_physical_ungrouped_aggregate`** (`UNGROUPED_AGGREGATE`) — runs per input
   batch, producing a **partial** aggregate (one row): `SUM` of this batch, `COUNT` of
   this batch, etc.
2. **`sirius_physical_ungrouped_aggregate_merge`** (`MERGE_AGGREGATE`) — collects all
   the partials and combines them into the **final** one row.

This file defines **both** classes. The pipeline splitter wires them with a `FULL`
barrier (merge waits for all partials), per the plan-gen doc. This local→merge shape
is *the* recurring pattern for blocking aggregations in Sirius.

## Shared setup: `build_aggregate_layout()` (114–229)

Both phases first translate DuckDB's bound aggregate expressions into a plan:

- Walks each `duckdb::BoundAggregateExpression`, branches on `function.name`:
  `count_star`, `count`, `sum`/`sum_no_overflow`, `min`, `max`, `avg`, `first`.
- For each, records an `aggregate_spec` (kind, input column index, return type) and
  the **cuDF merge kind** used in phase 2 (e.g. SUM→`cudf::aggregation::SUM`,
  first→`NTH_ELEMENT`).
- **AVG decomposition** (195–209): cuDF has no AVG reduce, so AVG becomes **two** local
  columns — a SUM and a COUNT — merged separately, then divided in phase 2. This is
  the same SUM+COUNT trick the plan-gen doc mentions.
- Rejects what the GPU path can't do: `DISTINCT` aggregates and multi-argument
  aggregates throw `not_implemented_exception` → CPU fallback.

## Phase 1 — local aggregate: `execute()` (267–391)

The `execute()` override, following the four-step shape from
[`sirius_physical_limit.md`](sirius_physical_limit.md). For each input batch, for each
`aggregate_spec`:
- `COUNT_STAR` → make a scalar = `view.num_rows()`.
- `COUNT` → `cudf::reduce(col, count_aggregation, INT64, stream)`.
- `SUM`/`MIN`/`MAX`/`AVG` → `cudf::reduce(col, {sum,min,max}_aggregation, out_type, stream)`.
- `FIRST` → `cudf::get_element(col, 0)`.

Each result is a 1-row column; the columns become a 1-row `cudf::table` wrapped as the
output batch. **Read the SUM widening (337–362) closely** — it's a great example of a
GPU correctness landmine: DECIMAL32→64→128 and small-int→INT64 casts *before* reducing
to avoid overflow. (Per operators.md, BIGINT SUM actually falls back to CPU entirely
because the GPU lacks an INT128 accumulator — the kind of footgun worth remembering.)

## Phase 2 — merge: `sirius_physical_ungrouped_aggregate_merge` (410–522)

- Constructed from the phase-1 operator (410/421), deep-copying the aggregate expressions.
- `execute()` (432): if one partial, pass through; else
  `gpu_merge_impl::merge_ungrouped_aggregate(partials, merge_kinds, ...)` combines them
  with the recorded cuDF merge kinds (`merge_ungrouped_aggregate`, 459). Then, if `has_avg`, `make_avg_column()` (231, called at 479)
  divides the merged SUM by the merged COUNT to produce the final average.
- **`get_next_task_input_data()` override (502)** — *this is the merge-specific bit
  worth seeing.* Instead of the base "one batch per port," it **drains every batch
  from the port** under the operator lock, so a single merge task sees *all* partials
  at once. That's how "merge everything" is expressed against the task machinery.

## How it closes the loop

For the trace query the full operator chain is: **GPU_SCAN** (read `lineitem`; post-`#871` —
this was `DUCKDB_SCAN` pre-`#871`) →
[grouped] **aggregate (local)** (partial SUM per batch) → **PARTITION/merge** (combine)
→ **RESULT_COLLECTOR** (materialize) → back to `sirius_interface::fetch_result_internal`
→ DuckDB. Reading this file you can now point at where the `SUM(l_quantity)` is
*actually computed on the GPU* (the `cudf::reduce` in phase-1 `execute`) and where the
partials become the final answer (phase-2 `merge`). That's the end of the Week 2 trace.

## Types fundamental to *this* file

- **`duckdb::BoundAggregateExpression`** *(DuckDB)* — a resolved aggregate call
  (`SUM(x)`, `AVG(y)`); `function.name`, `children`, `return_type`, `IsDistinct()`.
  **Think:** "one aggregate in the SELECT, fully resolved."
- **`cudf::reduce` + `cudf::reduce_aggregation`** *(cuDF)* — collapse a column to a
  scalar under an aggregation (SUM/MIN/MAX/COUNT). The GPU primitive doing the actual
  math. **Think:** "fold a column to one value on the GPU."
- **`cudf::aggregation::Kind`** *(cuDF)* — the enum naming merge operations, recorded in
  the layout for phase 2. **Think:** "how phase 2 recombines each partial column."
- **`aggregate_layout` / `aggregate_spec`** *(local to this file)* — the translated
  plan shared by both phases. **Think:** "the recipe mapping SQL aggregates → local
  columns + merge kinds."
- `pipelineable_operator_data`, `gpu_table_representation`, `cuda_stream_view` — see
  [`sirius_physical_operator.md`](sirius_physical_operator.md).

## Takeaway

The two-phase local→merge pattern here is the template for every blocking aggregation
(grouped included) and rhymes with the partition→merge sort path. With this, you've
traced a query from SQL string (`sirius_extension`) through lifecycle
(`sirius_interface`), build/launch (`sirius_engine`), translation (plan generator),
ownership (`sirius_context`), and down to the GPU `cudf::reduce` that computes the
answer. That's the Week 2 checkpoint — synthesized in
[`weeks/week2-concepts.md`](../../weeks/week2-concepts.md).
