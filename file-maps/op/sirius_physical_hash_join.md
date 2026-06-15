# `op/sirius_physical_hash_join.cpp` → Operator Map (the hard one)

Companion for **Week 5** of [`onboarding-path.md`](../../onboarding-path.md), tagged
**study** — at 1,224 lines, the most complex operator in Sirius. Pair it with CMU 15-445's
hash-join lecture and prime with
[`../../reference/explainers/hash-join-build-probe.md`](../../reference/explainers/hash-join-build-probe.md).
Read **after** the scheduler/creator maps and
[`sirius_physical_partition.md`](sirius_physical_partition.md) — partition is what feeds
this operator, and the join's task-creation overrides only make sense once you've seen the
hint chain ([`../creator/task_creator.md`](../creator/task_creator.md)).

> Source/doc paths relative to the Sirius repo root; study-note links relative to this
> file. Lines accurate as of 2026-06-15 — **re-grep**; this is a hot, large file.

## The key idea first: one operator, three execution modes

A hash join builds a hash table from one side (**build**) and probes it with the other
(**probe**). Sirius implements this **three different ways**, chosen at runtime, and the
single biggest thing to keep straight while reading is *which mode you're in*. The mode is
`HASH_JOIN_MODE` (hpp 51):

| Mode | When | How (cuDF) |
|---|---|---|
| **STANDARD** *(default)* | equality conditions; normal sizes | one-shot `cudf::inner_join` / `left_join` / `full_join` per partition pair (build+probe fused). |
| **BUILD_PROBE** | small build side that folds to a single batch | explicitly build a `cudf::hash_join` once, then probe it across many probe batches (reuses the table, pipelines better). |
| **MIXED_JOIN** | equality **and** inequality conditions (`a.x = b.x AND a.y < b.y`) | `cudf::mixed_inner_join` etc., passing the inequality as a cuDF **AST** from the translator. |

So the file is really three operators sharing setup. The mode is decided in
`update_join_exec_mode()` (cpp 411), called by the **partition** operator once it knows the
partition count and build-side bytes.

> **Updated for pull `d9172de6` (2026-06-15).** MARK joins are **now BUILD_PROBE-eligible**
> (`#927`) — see the correction in the Join-types section below. Line numbers refreshed; the
> three-mode structure is unchanged. Full diff in
> [`../../reports/post-pull-review-2026-06-15.md`](../../reports/post-pull-review-2026-06-15.md).

## Where this sits

`sirius_physical_hash_join` is a **`sirius_physical_partition_consumer_operator`** (hpp 54)
— it consumes partitioned data. The plan around a join is multi-pipeline:

```
build pipeline ─▶ PARTITION(build) ─┐
                                    ├─▶ HASH_JOIN   (this file: sink=build, source=probe)
probe pipeline ─▶ PARTITION(probe) ─┘
```

The `FULL` barrier on the build side (the hash table must be complete before any probe) is
the canonical reason `FULL` barriers exist — see
[`../../reference/explainers/hash-join-build-probe.md`](../../reference/explainers/hash-join-build-probe.md)
and the data-management doc. Partition keys/casts come from
[`sirius_physical_partition.md`](sirius_physical_partition.md); the inequality-AST for
MIXED_JOIN comes from
[`../expression_executor/gpu_expression_translator.md`](../expression_executor/gpu_expression_translator.md)
— both Week-3/4 threads converging here.

## Setup (constructors + condition analysis)

| Function (signature) | Line | Role |
|---|---|---|
| `static bool are_conditions_supported(conditions)` | 101 | gate: needs ≥1 equality condition; for mixed joins, equality and inequality columns must be disjoint per side (cuDF requirement). Failing → CPU fallback. |
| ctor `(op, left, right, cond, join_type, …projection maps…, pushdown)` | 188 | the full ctor; reorders conditions so equality keys come first (`num_equality_conditions`), computes per-key cast info (`key_casts`), sets `_join_mode = MIXED_JOIN` (cpp 323) when inequalities are present. |
| ctor `(op, left, right, cond, join_type, est_card, …)` | 327 | the slimmer ctor. |
| `void build_pipelines(current, meta)` | 405 | → `build_join_pipelines` (352): wires the build/probe pipelines + the `FULL` build barrier. |

**Key-cast subtlety (the same hashing footgun as PARTITION):** if the two sides of an
equality key have different physical widths (INT32 vs INT64), they must be cast to a common
type *before* hashing/comparison, or matching keys won't match. `key_casts` (hpp 199–207)
records this per key; `_built_table_cast_columns` keeps any cast build columns alive.

## Mode selection + the BUILD_PROBE state machine

`update_join_exec_mode(num_partitions, build_side_bytes, build_foldable)` (cpp 411) picks
BUILD_PROBE only when the build side is small **and** `build_foldable_to_single_batch` is
true (a downstream build-side CONCAT guaranteed one batch). Without that guarantee, the
runtime build-batch invariant would throw — so it stays STANDARD. This is exactly the
decision the partition sibling triggers (you saw the call site in
[`sirius_physical_partition.md`](sirius_physical_partition.md)).

BUILD_PROBE then runs a state machine, `BUILD_HASH_TABLE_STATE` (hpp 52):
`NOT_BUILT → SCHEDULING → SCHEDULED → BUILT (→ DESTROYED)`. The task-creation side of it:

| Function | Line | Role |
|---|---|---|
| `get_next_task_hint()` | 445 | tracks the build state; `READY` when both sides available (NOT_BUILT) or when probe data is available (BUILT). |
| `get_next_task_input_data_for_build_probe()` | 497 | SCHEDULING → pop **one build + one probe** batch; BUILT → pop **one probe** batch. The build/probe asymmetry. |
| `get_next_task_input_data()` | 563 | dispatches to the build_probe variant when in that mode, else the STANDARD partition×left×right grid walk. |

(This is the most elaborate `task_creator` override in the codebase — the payoff of
reading [`../creator/task_creator.md`](../creator/task_creator.md) first.)

## `execute()` — the three branches (cpp 885)

One method, branching on `_join_mode`. Read it branch-by-branch; don't try to read it
top-to-bottom as one flow.

- **BUILD_PROBE** (901–1014): on the build task, construct the structure — a
  `cudf::distinct_hash_join` (931) when build keys are proven unique, else a
  `cudf::hash_join` (938), or (for MARK) a `cudf::filtered_join` — and stash the build
  table (`_build_table`). On probe tasks, probe the stored structure. State transitions
  guard correctness (it throws on an invalid state, cpp 1009).
- **MIXED_JOIN** (1015–~1180): equality keys + the inequality **AST** drive
  `cudf::mixed_inner_join` (1059), `mixed_left_join` (1070/1090), `mixed_full_join` (1101),
  and the semi/anti variants `mixed_left_semi_join` (1049/1112/1135) /
  `mixed_left_anti_join` (1120/1150). Side-swapping handles RIGHT/OUTER.
- **STANDARD** (~1184–1278): one-shot per partition pair —
  `cudf::inner_join` (1206), `cudf::left_join` (1211/1216, swapped for RIGHT),
  `cudf::full_join` (1260), or a `cudf::distinct_hash_join` fast path (1186) when a side is
  unique. These return row-index pairs; the operator then `gather`s the output columns per
  the projection maps (`lhs_output_columns` / `rhs_output_columns` / `payload_columns`).

`on_finalize_operator()` (1279) releases the hash table / build table at end of life.

## Join types

`join_type` (`duckdb::JoinType`) selects INNER / LEFT / RIGHT / OUTER / SEMI / ANTI / MARK.
RIGHT/OUTER are typically implemented by swapping build/probe sides and using the LEFT/full
cuDF primitive (you see the `right_eq` / swapped-argument calls).

**MARK joins** output all left rows plus a BOOL8 "did it match?" column (cpp ~1231). As of
pull `d9172de6` they are **first-class in both modes**:
- **BUILD_PROBE MARK** (`#927`): `update_join_exec_mode` (411) lets MARK enter BUILD_PROBE,
  and `execute` builds a **persistent `cudf::filtered_join`** once on the right (filter) keys
  (`make_right_filtered_join_ptr`, cpp 81), probing it per batch — so the filter structure is
  reused across probe batches like a hash table.
- **STANDARD MARK** (`#924`): adaptively built on the *smaller* side
  (`make_right_filtered_join` chooses based on relative sizes, cpp ~93).

(Our earlier note that MARK "gates BUILD_PROBE off" is **no longer true** — that was the
pre-`#927` behavior.)

## Types fundamental to *this* file

- **`HASH_JOIN_MODE` / `BUILD_HASH_TABLE_STATE`** *(this file; hpp 51–52)* — the mode
  enum + the build-probe state machine. **Think:** "always know which of the three modes
  you're reading, and (in BUILD_PROBE) which state."
- **`cudf::hash_join` / `cudf::distinct_hash_join`** *(cuDF)* — the reusable hash table for
  BUILD_PROBE; `distinct_` is the unique-keys fast path. **Think:** "build once, probe
  many."
- **`cudf::filtered_join`** *(cuDF; added this pull)* — the reusable structure for **MARK**
  joins, built once on the right (filter) keys and probed per batch (`#927`). **Think:**
  "the MARK-join analogue of `hash_join`: build the filter once, test each probe batch."
- **`cudf::inner_join` / `left_join` / `full_join` / `mixed_*_join`** *(cuDF)* — the
  one-shot join primitives returning row-index pairs. **Think:** "the GPU join kernels;
  the operator's job is choosing + gathering."
- **`sirius::join_condition`** *(Sirius; holds a `sirius::ast::node`, not a DuckDB wrapper —
  changed this pull, `#701/#703`)* — one `left <cmp> right` clause with a Sirius-native
  `comparison_type`; reordered so equalities precede inequalities. **Think:** "one ON
  clause; equality keys hash, inequality keys feed the mixed-join AST."
- **`key_cast_info`** *(this file; hpp 199)* — per-key cast-before-compare flags. **Think:**
  "align widths so equal values hash equal" (same footgun as PARTITION).
- **`join_projection_columns`** *(this file; hpp 58)* — which columns/types each side
  outputs; drives the post-join gather. **Think:** "the SELECT list, split by side."
- `sirius_physical_partition_consumer_operator`, `read_only_data_batch`,
  `cuda_stream_view` — operator plumbing; see
  [`sirius_physical_operator.md`](sirius_physical_operator.md).

## Takeaway

The hash join is "hard" not because any one path is deep but because **three modes +
build/probe asymmetry + join-type/side-swapping + key casts** all live in one file. The
way to tame it: fix the mode first (`update_join_exec_mode`), then read only that branch of
`execute()`, keeping the partition feeder and the BUILD_PROBE state machine in mind.
Reaching this point — scan in, scheduled across GPUs without races, joined on the GPU —
is the Weeks 4–5 checkpoint, synthesized in
[`../../weeks/week4-5-concepts.md`](../../weeks/week4-5-concepts.md).
