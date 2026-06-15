# `op/sirius_physical_partition.cpp` → Operator Map (medium operator)

Companion for Week 3, **Day 3** of [`onboarding-path.md`](../../onboarding-path.md) — the
other "medium operator" choice, tagged **read closely**. This one is harder than
[`sirius_physical_top_n.md`](sirius_physical_top_n.md): it's less about a cuDF kernel and
more about how Sirius **splits data across partitions to feed joins and grouped
aggregates** — so it previews the Week 4–5 scheduling/multi-pipeline material. Pair with
`test/cpp/operator/test_physical_partition.cpp` (and `test_gpu_partition_impl.cpp`).

> Source/doc paths are relative to the Sirius repo root; study-note links are relative to
> this file. Lines accurate as of 2026-06-15 — re-grep if moved.

## The key idea first: PARTITION is the fan-out that makes joins/group-bys parallel

A hash join or grouped aggregate only works if **rows with the same key land together**.
PARTITION is the operator that arranges that: it hashes each row's key columns and routes
it to one of `N` partitions, so a downstream `MERGE_GROUP_BY` or partitioned hash join
can process each partition independently (and on different GPUs). It is both a
**source and a sink** (71–73) — it consumes a pipeline's output and re-emits it split by
partition into the next operators' ports.

So unlike TOP-N (self-contained compute), PARTITION's interesting surface is **control**:
how it learns its partition keys, how it decides `N`, and how it coordinates a
build/probe sibling pair. The actual hashing is delegated to
`gpu_partition_impl::hash_partition` (`op/partition/gpu_partition_impl.hpp`).

## Where this sits

- **Upstream:** whatever pipeline produces the rows to be partitioned.
- **Downstream:** a *partition consumer* — `MERGE_GROUP_BY`
  ([`sirius_physical_grouped_aggregate.md`](sirius_physical_grouped_aggregate.md)),
  hash join, or `MERGE_TOP_N` — which pulls one partition at a time. The `sink()` (206)
  pushes each partition's batch into the consumer's matching partition slot.
- **Sibling:** in a join, the build-side and probe-side PARTITION ops are linked
  (`_sibling_partition_op`) and must agree on `N`.

This is your first real look at the **task-hint / scheduling** surface that Weeks 4–5
cover in depth — treat the scheduling methods below as a preview, not something to fully
master yet.

## How it learns what to partition on: `get_partition_keys_and_type()` (75)

Called from the constructor. It inspects the **parent operator** to derive the partition
keys and type — a nice illustration of how an operator is configured by its place in the
plan:

- **HASH_JOIN** (78) → `PartitionType::HASH`; pull the equality/`not_distinct_from`
  conditions, extract the `BOUND_REF` column index on the build or probe side. **Subtle
  bit (93–110):** if a join condition has a `BOUND_CAST` (e.g. INT32 key vs INT64 key),
  it records the cast type — because cuDF's murmur3 hashes the *same* integer differently
  in different widths, so without aligning types, matching keys would scatter to
  different partitions. Worth reading; it's a genuine GPU-hashing footgun.
- **HASH_GROUP_BY / MERGE_GROUP_BY** (116 / 132) → `HASH`; keys = the aggregate's output
  grouping indices.
- **NESTED_LOOP_JOIN** (113) → `PartitionType::NONE`, `N = 1` (no key to hash on).
- **CONCAT** (137) → recurse into the concat's parent (the partition sits behind a
  concat).

## How it decides `N`: `determine_num_partitions()` (230) + the sibling dance

- `determine_num_partitions` (230) sums the input bytes and divides by
  `s_partition_size` (the target bytes/partition) to pick `N`, with a **multi-GPU floor**
  (253): if the input is big enough, force at least `num_gpus` partitions so every GPU
  gets work — but tiny tables stay at `N=1` to avoid cross-device overhead.
- `get_next_task_input_data()` (295) is where it gets genuinely concurrent. For a
  build/probe pair it takes a **`std::scoped_lock` over both** this and the sibling's
  mutex (302) to avoid ABBA deadlock, then (if not yet set) determines `N` once and
  writes it to *both* siblings (337–338). It also decides the join's `BUILD_PROBE`
  execution mode and flips the build-side CONCAT's `concat_all` — i.e. the partition
  count drives downstream join strategy. Read this for *why the locking is shaped this
  way*, not to memorize the join-mode logic (that's Week 4–5).
- `get_next_task_hint()` (265): a probe-side partition waits on its build sibling's hint
  until `N` is known, then behaves like a pipeline operator. This is the
  scheduling/"hint chain" the task-creator doc explains — a preview.

## The actual work: `execute()` (155) and `sink()` (206)

- `execute()` (155): expects one input batch; if `N < 2` or no keys, pass through (173).
  Otherwise dispatch on `_partition_type`: `HASH` → `gpu_partition_impl::hash_partition`
  (180, passing the cast types), `EVENLY` → `evenly_partition` (190), `NONE` → clone
  (193); `RANGE`/`CUSTOM` throw (not implemented). Returns one batch per partition.
- `sink()` (206): for each partition batch, push it into the downstream partition
  consumer's matching partition slot via `push_data_batch_partitioned` (220). This is the
  cross-pipeline handoff — the wiring `docs/super-sirius/data-management.md` (Week 5)
  formalizes with ports/repos.

## Reading it with the test

`test_physical_partition.cpp` and `test_gpu_partition_impl.cpp` exercise the hashing and
routing directly. Because this operator is so plan-shaped (its behavior depends on the
parent op), the tests are the clearest way to see it in isolation.

## Types fundamental to *this* file

- **`PartitionType`** *(this file; `{HASH, RANGE, EVENLY, NONE, CUSTOM}`)* — how rows are
  assigned to partitions; only HASH/EVENLY/NONE are implemented. **Think:** "the routing
  rule."
- **`gpu_partition_impl::hash_partition`** *(Sirius; `op/partition/`)* — the GPU kernel
  that hashes key columns and scatters rows into `N` output batches. **Think:** "the
  fan-out primitive."
- **`task_creation_hint` / `TaskCreationHint`** *(Sirius scheduling)* — what an operator
  tells the scheduler about readiness (`READY`, `WAITING_FOR_INPUT_DATA`). Met fully in
  Week 4. **Think:** "the operator's message to the scheduler."
- **`_sibling_partition_op` / `_is_build`** *(this file)* — the build/probe pairing for
  joins; both siblings must agree on `N`. **Think:** "the two halves of a join's
  partitioning, kept in lockstep."
- `pipelineable_operator_data`, `memory_space`, `cuda_stream_view`, ports — operator I/O
  plumbing; see [`sirius_physical_operator.md`](sirius_physical_operator.md).

## Takeaway

PARTITION is where "operator" starts meaning "participant in a parallel schedule," not
just "a cuDF call." Its core ideas — derive keys from the parent op, size `N` from bytes
(with a multi-GPU floor), coordinate a build/probe sibling pair under careful locking, and
route partitions into downstream consumer slots — are exactly the concurrency themes
Weeks 4–5 develop. If you want the gentler Day-3 read first, do
[`sirius_physical_top_n.md`](sirius_physical_top_n.md); if you're ready to peek at the
hard core early, this is the on-ramp.
