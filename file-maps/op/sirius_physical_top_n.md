# `op/sirius_physical_top_n.cpp` → Operator Map (medium operator)

Companion for Week 3, **Day 3** of [`onboarding-path.md`](../../onboarding-path.md) — the
"a medium operator" slot, tagged **read closely**. The plan offers a choice between this
and [`sirius_physical_partition.md`](sirius_physical_partition.md); TOP-N is the gentler
of the two (self-contained cuDF compute, no partitioning/scheduling subtlety), so start
here. Pair it with its test, `test/cpp/operator/test_physical_top_n.cpp`, and the
ORDER BY background:
[`../../reference/explainers/sorting-and-order-by.md`](../../reference/explainers/sorting-and-order-by.md).

> Source/doc paths are relative to the Sirius repo root; study-note links are relative to
> this file. Lines accurate as of 2026-06-15 — re-grep if moved.

## The key idea first: TOP-N is a two-phase local→merge, like aggregation

`SELECT … ORDER BY k LIMIT n` does **not** need to fully sort the whole table — it only
needs the `n` smallest/largest by `k`. And because a pipeline runs in parallel over many
batches, TOP-N uses the *same* two-phase shape you saw in
[`sirius_physical_ungrouped_aggregate.md`](sirius_physical_ungrouped_aggregate.md):

1. **`sirius_physical_top_n`** (`TOP_N`) — per input batch, keep that batch's top
   `offset + limit` rows.
2. **`sirius_physical_top_n_merge`** (`MERGE_TOP_N`) — concatenate the per-batch
   survivors, take the global top `offset + limit`, then drop the first `offset`.

Both classes are defined **in this one `.cpp`** (the merge's header is
`op/sirius_physical_top_n_merge.hpp`). The shared compute is one file-local helper,
`compute_top_n_table()` (44–136), called by both phases. Reading this file you can hold
the whole operator — phase 1, phase 2, and the kernel choice — in your head at once.

## The shared kernel: `compute_top_n_table()` (44)

This is the part worth reading slowly — it's a clean example of *picking the right cuDF
primitive for the case*, including a correctness fallback:

- **Empty / `limit == 0`** → empty table (54, 59).
- `keep_rows = min(num_rows, offset + limit)` (57) — how many to keep this phase.
- **Single key, no nulls** (87) → `cudf::top_k_order` (89) to get the top-k row indices,
  `cudf::gather` (90) to pull those rows, then `cudf::sort_by_key` (93) because
  `top_k_order` does **not** return them sorted.
- **Single key, nullable** (75) → **fallback**: `top_k_order` can't honor SQL
  `NULLS FIRST/LAST` (it treats NULLs as largest), so do a full `cudf::sort_by_key`
  honoring null placement (79), then `cudf::slice` the top `keep_rows` (84). A textbook
  "the fast primitive can't express the SQL semantics, so degrade to the correct one"
  comment — exactly the kind of GPU footgun the methodology wants you noticing.
- **Multi-key** (100) → no top-k shortcut; full `sort_by_key` over all keys then slice.

Order/null-order are translated from DuckDB's `BoundOrderByNode` via `to_cudf_order` /
`to_cudf_null_order` (`op/cudf_sort_order.hpp`). Only `BOUND_REF` ordering keys are
supported — anything else throws `not_implemented_exception` (→ CPU fallback).

## Phase 1 — `sirius_physical_top_n::execute()` (158)

Follows the same four-step operator shape as
[`sirius_physical_limit.md`](sirius_physical_limit.md):
1. downcast `operator_data` → `pipelineable_operator_data`, grab the read-only batch (162);
   TOP-N expects **exactly one** input batch per call (172).
2. get the `cudf::table_view` out of the batch's `gpu_table_representation` (182).
3. `compute_top_n_table(...)` with the batch's allocator (184).
4. wrap the result table as a new `gpu_table_representation` → `data_batch` → output
   (192–197). Note the **STREAM-LINEAGE** comment (189): the constructor records the
   writer event on `stream` so cross-device readers honor producer→consumer ordering — a
   small but real bit of the async/multi-GPU correctness machinery.

## Phase 2 — `sirius_physical_top_n_merge::execute()` (228)

- Constructed from the phase-1 operator (200), deep-copying `orders` (`copy_orders`).
- Collects **all** input batches' table views (258), `cudf::concatenate`s them (273) —
  this is where the per-batch survivors meet — then runs the same
  `compute_top_n_table()` over the combined table (276).
- **Applies `offset` here, once** (278–286): slice off the first `offset` rows of the
  global result (phase 1 deliberately keeps `offset + limit` so no qualifying row is lost
  early). Getting offset handling at the *merge* and not per-batch is the subtle
  correctness point.
- **`get_next_task_input_data()` override (299)** — like the aggregate merge, it **drains
  every batch** from its port under the lock so one merge task sees all partials at once
  (vs. the base "one batch per task"). This is the recurring "merge = collect everything"
  idiom against the task machinery.

## Reading it with the test

`test/cpp/operator/test_physical_top_n.cpp` is the fast way to *run* this in your head:
it builds input tables, drives `execute`, and checks the kept rows. A good first-PR
surface too — extending supported cases (e.g. an ordering expression beyond `BOUND_REF`)
or adding a test case is exactly the Week 3 Days 4–5 "ship something small" task.

## Types fundamental to *this* file

- **`duckdb::BoundOrderByNode`** *(DuckDB)* — one `ORDER BY` key: an `expression`, a sort
  `type` (asc/desc), and a `null_order`. **Think:** "one ORDER BY column + its
  direction + null placement."
- **`cudf::top_k_order` / `cudf::sort_by_key` / `cudf::gather` / `cudf::slice` /
  `cudf::concatenate`** *(cuDF)* — the GPU primitives this operator composes. **Think:**
  "the toolbox; the operator's job is choosing among them per case."
- **`sirius_physical_top_n_merge`** *(this file)* — the phase-2 partner; same fields,
  different `execute`. **Think:** "the global combiner; phase 1's results funnel into it."
- `pipelineable_operator_data`, `gpu_table_representation`, `cuda_stream_view`,
  `memory_space` — the operator I/O plumbing, see
  [`sirius_physical_operator.md`](sirius_physical_operator.md).

## Takeaway

TOP-N is the cleanest "medium" operator: a two-phase local→merge (familiar from
aggregation) whose interest is the **per-case cuDF primitive selection** in
`compute_top_n_table`, including a correctness fallback for nullable keys and the
offset-at-merge subtlety. With an operator like this read end-to-end and its test
understood, you're set up for the Days 4–5 first contribution. The harder Day-3
alternative, which pulls in partitioning + scheduling, is
[`sirius_physical_partition.md`](sirius_physical_partition.md).
