# `op/sirius_physical_partition.cpp` → Operator Map (medium operator, in depth)

Companion for Week 3, **Day 3** of [`onboarding-path.md`](../../onboarding-path.md) — the other
"medium operator" choice, tagged **read closely**. Harder than
[`sirius_physical_top_n.md`](sirius_physical_top_n.md): it's less about a cuDF kernel and more
about how Sirius **splits data across partitions to feed joins and grouped aggregates**, so it
previews the Week 4–5 scheduling/multi-pipeline material. Pair with
`test/cpp/operator/test_physical_partition.cpp` (and `test_gpu_partition_impl.cpp`), and read it
next to the converter that *inserts* it
([`../pipeline/sirius_pipeline_converter.md`](../pipeline/sirius_pipeline_converter.md), the
`split_join_sink` / `link_join_partition_siblings` phases).

> Source/doc paths are relative to the Sirius repo root; study-note links are relative to this
> file. Lines accurate as of 2026-06-17 — re-grep if moved.

## The key idea first: PARTITION is the fan-out that makes joins/group-bys parallel

A hash join or grouped aggregate only works if **rows with the same key land together**.
PARTITION arranges that: it hashes each row's key columns and routes it to one of `N` partitions,
so a downstream `MERGE_GROUP_BY` or partitioned hash join can process each partition
independently — and **on different GPUs**. It is both a **source and a sink** (`is_source` 71 /
`is_sink` 73), which is the structural thing to understand first (next section).

Unlike TOP-N (self-contained compute), PARTITION's interesting surface is **control**: how it
learns its partition keys (from the parent op), how it decides `N` (from input bytes, with a
multi-GPU floor), and how it coordinates a build/probe **sibling pair** under careful locking. The
actual hashing is delegated to `gpu_partition_impl::hash_partition` (`op/partition/`).

## Where this sits

- **Inserted by the converter.** PARTITION ops don't come from the plan — `split_join_sink`,
  `split_group_aggregate_sink`, `split_intermediate_joins`, and `split_delim_join_sink` mint them
  (see [`../pipeline/sirius_pipeline_converter.md`](../pipeline/sirius_pipeline_converter.md)).
  The build/probe pair is then linked (`set_sibling_partition_op`) by
  `link_join_partition_siblings`, and `configure_partition_min_partitions` sets the multi-GPU floor.
- **Upstream:** whatever pipeline produces the rows to partition.
- **Downstream:** a *partition consumer* — a hash join,
  `MERGE_GROUP_BY` ([`sirius_physical_grouped_aggregate.md`](sirius_physical_grouped_aggregate.md)),
  etc. — a `sirius_physical_partition_consumer_operator` that exposes
  `push_data_batch_partitioned(port, batch, partition_idx)`. Per the `set_min_num_partitions`
  contract (hpp 91–95), the consumer pins partition `i` to device `i % num_gpus` — which is *why*
  PARTITION forces a floor of `num_gpus` partitions on big inputs.
- **Sibling:** in a join, the build-side and probe-side PARTITION ops are linked
  (`_sibling_partition_op`) and must agree on `N`.

Treat the scheduling methods below as a Week 4–5 preview, not something to fully master yet.

## The structural crux: `execute()` *and* `sink()` both run, in one task

PARTITION is the **sink** of its (short) pipeline, so after the converter's
`finalize_pipeline_structure` it sits at `operators.back()`. Recall the task runtime
([`../pipeline/gpu_pipeline_task.md`](../pipeline/gpu_pipeline_task.md)): `compute_task` runs every
operator's `execute()` *including the sink's*, then `publish_output` calls the sink's `sink()`. So
for PARTITION **both halves fire in sequence within one task**:

1. **`execute()` (155) computes the partitioning** — takes the single input batch and produces a
   vector of `N` batches (one per partition). This is the "source" face: it *produces* partitioned
   data as the pipeline's output.
2. **`sink()` (205) routes those `N` batches** — pushes batch `i` into the downstream consumer's
   partition slot `i`. This is the "sink" face: it *places* the data where the consumer expects it.

That `execute` (compute) → `sink` (route) split, on the *same* operator, is the whole reason
PARTITION is both source and sink. Keep it in mind reading the rest.

## How it learns what to partition on: `get_partition_keys_and_type()` (75)

Called from the constructor (66). It inspects the **parent operator** to derive the partition keys
+ type — a clean example of an operator configured by its place in the plan:

| Parent op | type | keys |
|---|---|---|
| `HASH_JOIN` (78) | `HASH` | the equality / `not_distinct_from` conditions' `BOUND_REF` indices (build side → right, probe side → left) |
| `HASH_GROUP_BY` (115) / `MERGE_GROUP_BY` (131) | `HASH` | the aggregate's output grouping indices |
| `NESTED_LOOP_JOIN` (112) | `NONE`, `N=1` | no key to hash on |
| `CONCAT` (136) | — | recurse into the concat's parent (partition sits behind a concat); re-derives `_is_build` from `is_build_concat()` |

Two details worth reading:

- **The `extract_bound_ref_index` helper (39)** unwraps a `BOUND_REF` directly *or* through a
  `BOUND_CAST`. Note the join conditions are now `sirius::ast` nodes (post-PIMPL-retirement), so
  the code calls `sirius::ast::to_duckdb` (88–89) to convert them back to DuckDB expressions just
  to inspect their shape.
- **The cast-alignment footgun (93–110).** If a join condition has a `BOUND_CAST` (e.g. INT32 key
  vs INT64 key), it records the cast type into `_partition_key_cast_types`. cuDF's murmur3 hashes
  the *same* integer value differently at different widths — so without casting both sides to one
  type before hashing, matching keys would scatter to different partitions and the join would miss
  rows. An entry of `type_id::EMPTY` means "hash as-is." A genuine GPU-hashing correctness bug
  this code exists to prevent.

## How it decides `N`: `determine_num_partitions()` (229)

Reads the `"default"` input port's repository, **sums the bytes of all queued batches** (237–245),
and picks `N = max(1, total_bytes / s_partition_size)` (246, `s_partition_size` =
`hash_partition_bytes`, the target bytes/partition). Then the **multi-GPU floor** (252–254): if
`_min_num_partitions > 1` *and* `total_bytes >= _small_table_bytes`, raise `N` to at least
`_min_num_partitions` (= `num_gpus`) so every GPU gets a partition. Tiny tables stay at their
natural (small) `N` to avoid cross-device overhead. Returns `(N, total_bytes)`.

## The sibling dance: `get_next_task_input_data()` (294) and `get_next_task_hint()` (264)

This is where PARTITION stops being an operator and becomes a *scheduling participant*.

**`get_next_task_input_data()` (294)** — decides `N` exactly once, and (for joins) lets `N` drive
the join's execution mode:

- If there's a sibling, take a **`std::scoped_lock` over both** mutexes (301) — `scoped_lock`
  locks them together to avoid the ABBA deadlock two threads would hit grabbing one lock each.
- If `N` isn't set yet: `determine_num_partitions()`, then compute **`build_foldable`** =
  "does either sibling have a build-side `CONCAT` downstream?" (313–321). This gates BUILD_PROBE
  mode: that mode needs the build side to deliver exactly one batch at runtime, which only the
  build-side CONCAT with `concat_all` guarantees — without this gate a small build with no CONCAT
  would enter BUILD_PROBE and throw later.
- `hash_join.update_join_exec_mode(num_parts, total_bytes, build_foldable)` (322) — **the
  partition count drives the join strategy** (STANDARD vs BUILD_PROBE — see
  [`sirius_physical_hash_join.md`](sirius_physical_hash_join.md)). If BUILD_PROBE is chosen, flip
  the build-side CONCAT's `concat_all` on (326–334).
- Write `N` to **both** siblings (336–337) so build and probe partition identically.
- The non-sibling branch (group-by) just determines `N` alone (348–357).
- Either way, it then delegates to `sirius_physical_operator::get_next_task_input_data()` (359).

**`get_next_task_hint()` (264)** — the readiness signal the task-creator's hint chain follows
([`../creator/task_creator.md`](../creator/task_creator.md)):

- **probe side, `N` unknown** → return the **build sibling's** hint (272): the probe can't make
  tasks until the build has sized the partitioning.
- **probe side, `N` known** → behave as a normal pipeline op: `READY` if the input port has data,
  `WAITING_FOR_INPUT_DATA` (with the upstream producer) if the source pipeline isn't finished,
  else `nullopt` (277–288).
- **otherwise** → the base operator hint (290).

## The work: `execute()` (155) and `sink()` (205)

- **`execute()` (155)** — expects exactly one input batch and a set `_num_partitions` (else
  throws). If `N < 2` or no keys → **pass through** the input unchanged (172–174). Otherwise
  dispatch on `_partition_type`:

  | type | action |
  |---|---|
  | `HASH` (178) | `gpu_partition_impl::hash_partition(batch, keys, cast_types, N, stream, space)` |
  | `EVENLY` (188) | `gpu_partition_impl::evenly_partition(batch, N, …)` |
  | `NONE` (192) | clone the single batch (no real partitioning) |
  | `RANGE` / `CUSTOM` | throw — **not implemented** |

  Returns a `pipelineable_operator_data` of `N` batches.
- **`sink()` (205)** — for each partition batch `i`, for each `next_port_after_sink`, cast the
  downstream op to `sirius_physical_partition_consumer_operator` and call
  `push_data_batch_partitioned(port_name, batch, i)` (219). Throws if the downstream isn't a
  partition consumer. This is the cross-pipeline handoff the ports/repos machinery
  (`docs/super-sirius/data-management.md`, Week 5) formalizes.

## Memory estimate

`no_history_peak_memory_estimate()` (362): `N == 1` → `0` (pure passthrough, no extra memory);
otherwise `2 × input bytes` (partitioning roughly doubles peak — the input plus the scattered
copies). Feeds the task's reservation sizing
([`../pipeline/gpu_pipeline_task.md`](../pipeline/gpu_pipeline_task.md)).

## Member fields (hpp)

| Field | Meaning |
|---|---|
| `_parent_op` | the op that determines the keys (HASH_JOIN / GROUP_BY / CONCAT…) |
| `_hash_join_op` | set only when the parent is a hash join; receives `update_join_exec_mode` |
| `_sibling_partition_op` | the build/probe partner; both must agree on `N` |
| `_is_build` | which side of the join this partition feeds |
| `_partition_keys` | column indices to hash on |
| `_partition_key_cast_types` | per-key cast for hash alignment (`EMPTY` = as-is) |
| `_num_partitions` | `optional<int>` — set once, in `get_next_task_input_data` |
| `_partition_type` | `{HASH, RANGE, EVENLY, CUSTOM, NONE}` |
| `s_partition_size` | target bytes per partition (drives `N`) |
| `_min_num_partitions` / `_small_table_bytes` | multi-GPU floor + the threshold below which it's ignored |

## Types fundamental to *this* file

- **`PartitionType {HASH, RANGE, EVENLY, CUSTOM, NONE}`** *(hpp 31)* — the routing rule; only
  HASH/EVENLY/NONE are implemented (RANGE/CUSTOM throw). **Think:** "how rows are assigned to
  partitions."
- **`gpu_partition_impl::hash_partition`** *(Sirius; `op/partition/`)* — hashes key columns
  (applying the cast types) and scatters rows into `N` output batches. **Think:** "the fan-out
  primitive."
- **`sirius_physical_partition_consumer_operator`** *(Sirius op)* — the downstream base
  (hash join, merge_group_by) with `push_data_batch_partitioned`. **Think:** "who receives a
  partitioned batch, and pins it to a GPU."
- **`task_creation_hint` / `TaskCreationHint`** *(Sirius scheduling)* — `{READY,
  WAITING_FOR_INPUT_DATA}` + producer; the operator's readiness message. Met fully in Week 4
  ([`../creator/task_creator.md`](../creator/task_creator.md)). **Think:** "what the operator tells
  the scheduler."
- **`_sibling_partition_op` / `_is_build`** *(this file)* — the build/probe pairing; both siblings
  must agree on `N`. **Think:** "the two halves of a join's partitioning, kept in lockstep."
- `pipelineable_operator_data`, `memory_space`, `cuda_stream_view`, ports — operator I/O plumbing;
  see [`sirius_physical_operator.md`](sirius_physical_operator.md).

## Takeaway

PARTITION is where "operator" starts meaning "participant in a parallel schedule." Its core ideas:
it's **both source and sink** (`execute` computes the `N`-way fan-out, `sink` routes each partition
to a consumer slot, both in one task); it **derives keys from the parent op** (with a real cuDF
murmur3 cast-alignment fix); it **sizes `N` from input bytes** with a multi-GPU floor that, via the
consumer's `i % num_gpus` pinning, spreads work across devices; and it **coordinates a build/probe
sibling pair** under a two-lock `scoped_lock`, where the partition count even **drives the join's
execution mode**. These are exactly the concurrency themes Weeks 4–5 develop. For the gentler Day-3
read first, do [`sirius_physical_top_n.md`](sirius_physical_top_n.md); to see how PARTITION gets
created and paired, read [`../pipeline/sirius_pipeline_converter.md`](../pipeline/sirius_pipeline_converter.md).
