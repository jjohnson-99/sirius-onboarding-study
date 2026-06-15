# `op/sirius_physical_operator.cpp` → Base-Interface Map

Companion for Week 2, **Days 4–5** of [`onboarding-path.md`](../../onboarding-path.md),
tagged **read**. Read `src/op/sirius_physical_operator.cpp` (+ the header
`src/include/op/sirius_physical_operator.hpp`) with the **"Base Class"** section of
`docs/super-sirius/operators.md` open. Read this **before** the concrete operators —
it's the contract they all implement.

> Source/doc paths are relative to the Sirius repo root. Line
> numbers accurate as of 2026-06-10; re-grep if the file moved.

## Where this sits

Everything in `src/op/` derives from `sirius_physical_operator`. The plan generator
builds a tree of these; the engine arranges them into pipelines; the executors call
their `execute()`/`sink()`; the task creator calls their `get_next_task_hint()`. So
this one class is the seam between **plan**, **scheduling**, and **GPU work**. Learn
its four interfaces and every concrete operator becomes "fill in these blanks."

## Two class families in this file

### 1. `operator_data` — what flows *between* operators (header 79–254)

The batches passed in/out of operators. A small hierarchy:

- **`operator_data`** (base, 79) — opaque; just `get_type()`, `prepare_for_processing()`,
  `get_estimated_size_in_bytes()`.
- **`pipelineable_operator_data`** (155) — the common one: holds a vector of
  `cucascade::data_batch` (idle) and/or `read_only_data_batch` (locked) views, with
  **lazy conversion** between them. `prepare_for_processing()` (cpp 80) locks each
  batch into the task's reserved memory space before `execute()` runs. **This is the
  "Data Batch" wrapper from the glossary** — a wrapper around a `cudf::table` on the
  GPU.
- **`partitioned_operator_data`** (232) — adds a partition index, for
  PARTITION/CONCAT/MERGE operators.

**Think:** `operator_data` = the typed envelope around GPU columns moving through a
pipeline. `get_read_only_batches()` / `get_data_batches()` are how an operator's
`execute()` gets at the `cudf::table_view` inside.

### 2. `sirius_physical_operator` — the operator itself (header 272–513)

Core state: `type`, `children` (operators feeding *into* this one), `types` (output
column types), `operator_id` (auto-incremented), and a per-operator `lock`. Then four
virtual interfaces every operator selectively overrides.

## The four interfaces (this is the contract)

| Interface | Key methods | Base behavior (this file) |
|---|---|---|
| **Execute** | `execute(input_data, stream)` | cpp 248 — base is a **no-op** returning an empty batch. Concrete operators override to do GPU work (this is where the `cudf::` calls go). Called on *every* operator in a pipeline. |
| **Sink** | `sink(output_data, stream)`, `is_sink()` | cpp 238 — base `sink()` pushes each output batch to every `next_port_after_sink` via `push_data_batch`. Called only on the *last* operator of a pipeline. |
| **Source** | `is_source()`, `source_order()` | base `is_source()` = false. Scans/blocking operators override to true. |
| **Pipeline construction** | `build_pipelines(current, meta_pipeline)` | cpp 145 — the recursion that turns the operator tree into pipelines (see below). |
| **Task creation** | `get_next_task_hint()`, `get_next_task_input_data()`, `all_ports_empty()` | cpp 275/316/332 — port-readiness + batch-popping logic (see below). |

### `build_pipelines()` (cpp 145) — how the tree becomes pipelines

The base implements the common three cases (operators.md / plan-gen Part 2):
- **sink** (blocking op): set itself as the current pipeline's source, then spawn a
  *child* meta-pipeline for its single child (the build input).
- **source** (no children): set itself as the pipeline source.
- **streaming** (one child, not a sink): add itself as an intermediate op, recurse
  into the child.

Concrete operators (hash join, CTE, delim join) override this for their multi-child
shapes. Reading the base version is enough to follow the target query's straight-line
scan→aggregate pipeline.

### `get_next_task_hint()` (cpp 275) — port-readiness logic

This is the function the **task creator** polls to decide if an operator can run. It
walks the operator's input `port`s and applies the barrier rules:
- A **FULL** barrier port is not ready until its `src_pipeline` is finished (the
  blocking-operator rule).
- A **PIPELINE/PARTIAL** port is ready as soon as it has data.
- Returns `READY` (make a task) or `WAITING_FOR_INPUT_DATA` (with a pointer to the
  producer to chase). Returns `nullopt` when there's nothing to do.

The `port` struct (header 424) carries `{MemoryBarrierType type, shared_data_repository* repo,
src_pipeline, dest_pipeline}`. **This barrier logic is the heart of how Sirius knows
when an operator may execute** — worth reading slowly even though it's the base class.
Concrete operators with custom readiness (scans, sort_sample, merges) override it.

## How a concrete operator reads after this

Once you've got the base, every operator in `src/op/` is just: a constructor (build
state from the plan), an `execute()` override (one or more `cudf::` calls turning
input `table_view`s into output batches), and maybe overrides of `is_source()`,
`get_next_task_hint()`, or `get_next_task_input_data()`. That's exactly the shape of
the three you read next:
[`sirius_physical_limit.md`](sirius_physical_limit.md) (execute only),
[`sirius_physical_duckdb_scan.md`](sirius_physical_duckdb_scan.md) (source, custom
hint), [`sirius_physical_ungrouped_aggregate.md`](sirius_physical_ungrouped_aggregate.md)
(execute + custom drain).

## Types fundamental to *this* file

- **`operator_data` / `pipelineable_operator_data`** *(Sirius)* — the inter-operator
  data envelope. **Think:** the GPU batch wrapper; `get_read_only_batches()` →
  `cudf::table_view`.
- **`cucascade::data_batch` / `read_only_data_batch`** *(cuCascade)* — the actual
  tiered-memory batch and a read-locked view of it. Detail comes in Weeks 5–6
  (`data-management.md`, `memory-management.md`). **Think:** a `cudf::table` that may
  live on GPU/host/disk, plus a RAII read-lock.
- **`port` / `MemoryBarrierType` (PIPELINE/PARTIAL/FULL)** *(Sirius)* — the inter-pipeline
  edges and their synchronization semantics; the input to `get_next_task_hint()`.
  **Think:** "an inbox from an upstream pipeline, tagged with how long to wait."
- **`rmm::cuda_stream_view`** *(RMM)* — the CUDA stream every `execute()`/`sink()` runs
  on. **Think:** the GPU work queue this task uses.

## Takeaway

This file defines the verbs (`execute`/`sink`), the nouns (`operator_data`), and the
scheduling hooks (`get_next_task_hint` + ports/barriers) that the entire `op/` tree
shares. With it internalized, the concrete operators are short. Next: the smallest one.
