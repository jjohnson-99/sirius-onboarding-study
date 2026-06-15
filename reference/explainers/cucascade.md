# cuCascade (the tiered-memory & data-flow library)

If [`cudf.md`](cudf.md) is the library that gives Sirius GPU *compute* primitives,
**cuCascade** is the library that gives it *data residency* — tiered memory (GPU/host/disk),
spillable data batches, and the containers those batches flow through between pipelines.
It's the substrate behind [`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md): that
explainer is the *concepts*; this is the *library and its types*.

> **Prime around:** Week 5 (data batches & repositories — `docs/super-sirius/data-management.md`)
> → Week 6 (memory tiers & spilling — `docs/super-sirius/memory-management.md`). cuCascade
> is an NVIDIA library, vendored as a submodule.

## What it is

cuCascade is a C++ library for **tiered GPU memory management**: it lets data live on the
**GPU**, in **pinned host** RAM, or on **disk**, and move between those tiers, while staying
usable by GPU code. It sits *on top of* RMM (allocation) and CUDA streams (movement), and
*around* cuDF (the data it holds is usually a `cudf::table`). Sirius leans on it for two
jobs: **coping with data bigger than GPU memory** (spilling) and **passing data between
pipelines** (repositories).

## Core abstractions

| Type | What it is |
|---|---|
| **`memory::memory_space`** (+ **`Tier`** = GPU/HOST/DISK) | a tiered allocation domain — typically one per GPU (NUMA-aware). Reservations are made against a space; it hands out the allocator cuDF uses. |
| **`idata_representation`** → **`gpu_table_representation`**, **`host_data_representation`** | "the same logical data, in a given tier." The GPU one wraps a `cudf::table` (`get_table_view()`); the host one holds it in pinned RAM; disk is the deepest. |
| **`data_batch`** | the **movable/spillable unit**: a representation + a small **state machine** (idle ↔ read-only-locked) + a batch id. This is what flows through Sirius pipelines. |
| **`read_only_data_batch`** | an **RAII read-lock** on a batch — keeps it resident (un-spillable) while a task reads it; released on destruction. |
| **converter registry** (`representation_converter_registry`) | converts a batch **between tiers** on a stream (e.g. GPU→host) — the spill/promote mechanism. Sirius wraps it as `sirius::converter_registry`. |
| **`shared_data_repository`** (+ **`shared_data_repository_manager`**) | a container/queue of `data_batch`es **between pipelines** — `add_data_batch` / `pop_next_data_batch` / `total_size`. The manager owns them all. |

## How it relates to the other libraries

- **RMM** — allocation backend; a `memory_space` provides the RMM allocator cuDF uses.
  ([`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md))
- **cuDF** — the *contents*: `gpu_table_representation` wraps a `cudf::table`; operators get
  a `table_view` out of a batch to call `cudf::` on. ([`cudf.md`](cudf.md))
- **CUDA streams** — tier conversions (D2H/H2D copies) run on streams, and a `data_batch`
  records **stream lineage** (which stream/device wrote it) so cross-stream/GPU readers wait
  correctly. ([`cuda-streams-and-async.md`](cuda-streams-and-async.md))

## How Sirius uses it

- **The unit of data flow.** An operator's `operator_data` wraps cuCascade `data_batch`es;
  `execute()` locks them (`read_only_data_batch`), pulls a `cudf::table_view`, runs cuDF, and
  emits new batches. (See [`file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md).)
- **The ports between pipelines.** A `shared_data_repository` *is* the port a sink pushes
  into and a downstream task pops from — the buffer in the push-then-scheduled handoff of
  [`push-vs-pull-execution.md`](push-vs-pull-execution.md). It's also where idle intermediates
  sit, making them the spill candidates.
- **The reservation/spill domain.** Reservations are against a `memory_space`; spilling
  ("downgrade") **converts** a batch's representation GPU→host via the converter registry,
  freeing GPU memory; promotion converts it back. The downgraded host/disk form is a
  `spilling::allocation`.
- **Multi-GPU.** One `memory_space` per GPU (NUMA-local host tier), with stream-lineage +
  P2P coordinating cross-GPU movement.

## The framing

Two NVIDIA libraries do the heavy lifting under Sirius, with a clean split:

- **cuDF** — *what to compute*: GPU relational primitives over `cudf::table`s.
- **cuCascade** — *where the data lives and how it moves*: tiers, spillable `data_batch`es,
  and the repositories they flow through.

Sirius is largely the layer that maps DuckDB plans onto cuDF calls (the operators) and wires
cuCascade batches/repositories/memory-spaces between them (the pipelines + scheduler). Keep
that division in mind and a lot of the codebase sorts itself into "cuDF call" vs. "cuCascade
data plumbing."

## See also

- [`gpu-memory-and-spilling.md`](gpu-memory-and-spilling.md) — the concepts (reservations,
  spilling, OOM) this library implements.
- [`cudf.md`](cudf.md) — the compute counterpart; [`cuda-streams-and-async.md`](cuda-streams-and-async.md)
  — the streams tier-conversions run on.
- [`push-vs-pull-execution.md`](push-vs-pull-execution.md) — repositories as the inter-pipeline
  ports.
