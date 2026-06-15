# Weeks 4–5 Concepts — Concurrency, Scheduling, and the Scan Subsystem

The Weeks 4–5 checkpoint in [`onboarding-path.md`](../onboarding-path.md):

> Understand how work is scheduled across threads/GPUs and how data flows between
> pipelines without races.

Weeks 2–3 answered *what runs* (the spine, the operators, expressions). Weeks 4–5 answer
*how it runs concurrently* — the genuinely hard core, where the bugs are races and leaks,
not wrong kernels. This is two weeks for a reason; don't rush it. The DB-theory anchor is
**Morsel-Driven Parallelism** (Leis et al., 2014) — prime with
[`../reference/explainers/morsel-driven-parallelism.md`](../reference/explainers/morsel-driven-parallelism.md)
and [`../reference/explainers/push-vs-pull-execution.md`](../reference/explainers/push-vs-pull-execution.md).

The synthesis trace for these weeks is a query **with a join** (the weekend exercise):

```sql
SELECT ... FROM lineitem JOIN orders ON l_orderkey = o_orderkey ...;
```

A join forces everything interesting: multiple pipelines, cross-pipeline data flow with
barriers, multi-GPU scheduling, and the build/probe split.

## Part 1 — The scheduler tier: three roles, three executors (Week 4)

**File maps:**
[`../file-maps/pipeline/task_scheduler.md`](../file-maps/pipeline/task_scheduler.md),
[`../file-maps/pipeline/gpu_pipeline_executor.md`](../file-maps/pipeline/gpu_pipeline_executor.md),
[`../file-maps/creator/task_creator.md`](../file-maps/creator/task_creator.md) ·
**doc:** `docs/super-sirius/pipeline-execution.md`, `task-creator.md`

In Week 2 the engine ([`../file-maps/sirius_engine.md`](../file-maps/sirius_engine.md))
built pipelines and handed them to "the scheduler" as a black box. Weeks 4–5 open that
box. The clean way to hold it is **three roles**:

- **`task_creator`** decides **what** runs next. Its core is a *hint-chain recursion*: an
  operator that isn't ready (`get_next_task_hint()` → `WAITING_FOR_INPUT_DATA`) points at
  its upstream producer, and the creator recurses until it finds the deepest *ready*
  operator — so data flows from the deepest producers first. It then builds the right task
  kind (scan task vs. GPU pipeline task) and dispatches it.
- **`task_scheduler`** decides **where** (which GPU) and **when**. One management thread
  runs a **pull-signal matcher**: GPUs announce a free worker (`device_ready`), and the
  scheduler pops a task whose device *preference* matches (or that has none). Preference is
  a correctness constraint — a task may reference data resident only on one GPU.
- **`gpu_pipeline_executor`** (one per GPU) **runs** it: reserve memory, acquire a stream,
  dispatch to a bounded worker pool, and — importantly — turn out-of-memory into a
  **resumable retry** (`oom_reschedule_exception` carries a resume operator index) rather
  than a crash.

These three, plus the host-I/O **`duckdb_scan_executor`** and the downgrade executor, all
inherit one base (`itask_executor`) — so "executor" means the same reserve→pop→dispatch
loop filled in five times.

> **Two honesty notes for these maps** (the friend will check): (1) the scheduler is now a
> *pull-signal* model — the doc's "SCHED-RR round-robin counter" is superseded, and the
> counter field survives only as a test shim; (2) `MAX_OOM_RETRIES` is **100** in code, not
> the doc's 10. Trust the code. Both are flagged in the respective file-maps.

The single most important correctness property is the **per-task-device contract**: before
any operator's `execute()` runs, the task layer locks-or-converts *every* input batch onto
the task's reservation device (`prepare_for_processing` → `lock_or_prepare_batch`). That's
the postcondition that lets operators safely read `batches[0]->get_memory_space()` — the
caveat you saw sprinkled through the Week-3 operator maps now has its source.

## Part 2 — Data flow between pipelines: batches, repos, ports, barriers (Week 5)

**doc:** `docs/super-sirius/data-management.md` (a *read*, no single source file)

Pipelines don't call each other; they hand off **data batches** through **ports** backed
by **data repositories** (thread-safe queues keyed by `(operator_id, port_id)`). The
concepts that matter:

- **`data_batch` 3-state lock model** (idle ↔ read_only ↔ mutable_locked): the idle
  `shared_ptr` grants *no* data access — you must take a RAII `read_only`/`mutable`
  accessor. This is how concurrent readers and the downgrade (spill) path coexist without
  races. Prime with
  [`../reference/explainers/nulls-and-validity.md`](../reference/explainers/nulls-and-validity.md)
  only for the column internals; the lock model itself is in the data-management doc.
- **Barriers** on ports decide *when* a downstream pipeline may consume:
  - **`PIPELINE`** — no sync, data flows within a pipeline;
  - **`PARTIAL`** — consume incrementally but respect pipeline boundaries;
  - **`FULL`** — wait until the upstream pipeline is *completely* finished. **The hash-join
    build side is the canonical `FULL` barrier** — the whole hash table must exist before
    probing.
- **`operator_data` hierarchy** — `operator_data` (empty base) → `pipelineable_operator_data`
  (batch vector + the `prepare_for_processing` you met in Part 1) →
  `partitioned_operator_data` (adds a partition index, for CONCAT / MERGE_* / the join).

This is the "without races" half of the checkpoint: locks on batches, barriers on ports.

## Part 3 — The source: scan subsystem (Week 5)

**File map:**
[`../file-maps/op/scan/duckdb_scan_executor.md`](../file-maps/op/scan/duckdb_scan_executor.md)
· **doc:** `docs/super-sirius/scan.md` (largest doc — read the executor, skim the rest)

Every query starts at a scan. The `duckdb_scan_executor` is the host-I/O executor: it runs
`duckdb_scan_task`s that pull DuckDB chunks into **column builders** → a
`host_data_representation`, self-scheduling until the source drains. Two things make it
more than "read rows": a **four-level result cache** (`NONE` / `PARQUET` / `TABLE_HOST` /
`TABLE_GPU`) that lets re-runs skip decompression or even data movement, and
**memory-proportional multi-GPU targeting** (with sticky batch→GPU affinity to keep
`TABLE_GPU` cache hits valid). Keep the scan_manager / `sirius::io` / Iceberg machinery as
skim-on-demand — the plan only asks you to *read the executor* and *skim the GPU parquet
decoders*.

## Part 4 — The capstone operator: hash join (Week 5)

**File map:**
[`../file-maps/op/sirius_physical_hash_join.md`](../file-maps/op/sirius_physical_hash_join.md)
· **prime:**
[`../reference/explainers/hash-join-build-probe.md`](../reference/explainers/hash-join-build-probe.md)
+ CMU's hash-join lecture

Everything converges on the join. It's the hardest file (1,224 lines) because it's really
**three operators in one** — `STANDARD` (one-shot `cudf::inner_join`/`left_join`/`full_join`
per partition pair), `BUILD_PROBE` (build a reusable `cudf::hash_join` once, probe across
batches, driven by a `NOT_BUILT→…→BUILT` state machine), and `MIXED_JOIN` (`cudf::mixed_*`
with the inequality passed as a cuDF AST from the translator). The taming strategy: **fix
the mode first**, then read only that branch of `execute()`.

The join is where the prior weeks' threads meet:
- **PARTITION** ([`../file-maps/op/sirius_physical_partition.md`](../file-maps/op/sirius_physical_partition.md))
  feeds it, decides its mode (`update_join_exec_mode`), and supplies aligned partition keys;
- the **translator**
  ([`../file-maps/expression_executor/gpu_expression_translator.md`](../file-maps/expression_executor/gpu_expression_translator.md))
  supplies the MIXED_JOIN inequality AST;
- the **`task_creator`** override (`get_next_task_input_data_for_build_probe`) is the most
  elaborate in the codebase — the build/probe asymmetry (first task needs both sides, later
  tasks only probe);
- the **`FULL` barrier** (Part 2) sequences build before probe.

## Where this leaves you

You can now trace a join query as *concurrent work*: scans land batches on GPUs
(cache-aware, load-balanced), the creator's hint chain turns ready operators into tasks,
the scheduler matches them to free GPUs honoring data-locality preference, the per-GPU
executors run them (spilling + retrying on OOM), and data crosses pipelines through
lock-protected batches and barrier-gated ports — culminating in the three-mode hash join.
That's the Weeks 4–5 checkpoint: **how work is scheduled across threads/GPUs and how data
flows between pipelines without races.** Weeks 6–7 go after memory management, multi-GPU,
fallback, and a non-trivial contribution.
