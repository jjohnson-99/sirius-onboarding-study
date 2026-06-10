# `op/sirius_physical_limit.cpp` → Operator Map (warm-up)

Companion for Week 2, **Days 4–5** of [`onboarding-path.md`](../../onboarding-path.md).
The plan calls this a **warm-up** (tiny, ~130 lines): the simplest operator that
actually *does* something on the GPU, so it's the cleanest first look at the
`execute()` pattern from [`sirius_physical_operator.md`](sirius_physical_operator.md).
Read with the `STREAMING_LIMIT` entry in `docs/super-sirius/operators.md`.

> Paths relative to the Sirius repo root (`../../sirius/`). Lines as of 2026-06-10.

## Where this sits

`sirius_physical_streaming_limit` (`STREAMING_LIMIT`) is a **streaming operator** — in
`build_pipelines` terms, it's added as an intermediate op and passes data through
without buffering. The plan generator produces it from `LOGICAL_LIMIT`
(`sirius_plan_limit.cpp`). Not in the target aggregate query, but the ideal first
`execute()` to read because it's pure and self-contained.

## The one idea: parallel LIMIT via atomic claim

Multiple tasks may run this operator concurrently (one per input batch), yet together
they must emit at most `LIMIT` rows after skipping `OFFSET`. The trick is two shared
atomic counters and a lock-free `claim()`:

| Member (header) | Role |
|---|---|
| `_remaining_offset` (atomic int64) | rows still to skip |
| `_remaining_limit` (atomic int64) | rows still allowed to emit |
| `_limit_exhausted` (atomic bool) | set when limit hits 0 → lets the pipeline finish early (`is_limit_exhausted()`) |

| Function | Line | Role |
|---|---|---|
| ctor | 31 | Reads constant LIMIT/OFFSET values into the atomics. |
| `int64_t claim(atomic<int64_t>& counter, int64_t max_claim)` | 55 | **The primitive.** CAS loop: atomically subtract up to `max_claim` from `counter`, return how much it actually got. Lock-free coordination across tasks. |
| `unique_ptr<operator_data> execute(input_data, stream)` | 68 | The `execute()` override. |

## Reading `execute()` (68–126) — the pattern to absorb

For each input batch:
1. Get the `cudf::table_view` via `input.get_read_only_batches()` →
   `cast<gpu_table_representation>().get_table_view()` (the standard way an operator
   reaches GPU columns — you'll see this exact idiom everywhere).
2. `claim()` from `_remaining_offset` to decide how many rows to skip, then `claim()`
   from `_remaining_limit` for how many to take.
3. `cudf::slice(view, {start, end}, stream)` to cut out that row range on the GPU.
4. Wrap the sliced `cudf::table` in a `gpu_table_representation` → `data_batch` →
   push into a `pipelineable_operator_data` output.

That four-step shape — **get table_view → call a `cudf::` function on `stream` →
re-wrap the result as a batch** — is the skeleton of *every* operator's `execute()`.
Limit just uses `cudf::slice`; the aggregate uses `cudf::reduce`; a filter uses the
expression executor. Internalize it here where there's nothing else going on.

## Types fundamental to *this* file

- **`duckdb::BoundLimitNode`** *(DuckDB)* — the bound LIMIT/OFFSET value (constant or
  expression). Here only `CONSTANT_VALUE` is supported; non-constant throws
  `not_implemented_exception`. **Think:** "the resolved N in `LIMIT N`."
- **`cudf::slice`** *(cuDF)* — zero-copy row-range view of a table. **Think:** GPU
  `[start:end)` without copying data.
- Everything else (`pipelineable_operator_data`, `gpu_table_representation`,
  `cuda_stream_view`) is in [`sirius_physical_operator.md`](sirius_physical_operator.md)
  / [`duckdb-types-glossary.md`](../../reference/duckdb-types-glossary.md).

## Takeaway

This is the `execute()` template in miniature. Carry the four-step shape into the next
two operators — the scan (which *produces* batches as a source) and the aggregate
(which *reduces* them, in two phases).
