# Morsel-Driven Parallelism

> **Prime around:** Week 4. Read alongside the Leis et al. (2014) paper the
> [onboarding plan](onboarding-path.md) schedules for Weeks 4–5 — it's the design
> philosophy behind Sirius's `task_scheduler` / `task_creator`, which you read that week.

Once an engine is columnar, vectorized, and push-based, one question remains: **how do
you spread the work across many cores (or GPUs) without one slow partition dragging the
whole query down?** Morsel-driven parallelism is the modern answer, and it's the
philosophy Sirius's task scheduler is built on. It's the parallelism layer on top of
[`vectorized-execution.md`](vectorized-execution.md) and
[`push-vs-pull-execution.md`](push-vs-pull-execution.md).

## The old way: exchange-operator (plan-driven) parallelism

Classic parallel engines decide parallelism **at plan time**. The optimizer picks a
degree of parallelism *N* and inserts *exchange* operators that partition data between
*N* fixed thread-groups (e.g. hash-partition the rows, one partition per thread). Each
thread runs its own copy of the plan over its partition.

The problems:
- **Skew = stragglers.** If one partition is bigger or slower, its thread finishes last
  and everyone waits. Parallelism is only as fast as the unluckiest partition.
- **Fixed too early.** *N* is baked in at plan time; it can't adapt to runtime load or to
  other queries competing for cores.
- **NUMA-oblivious.** Threads and data aren't co-located on the same memory node.

## The morsel-driven idea: data-driven scheduling + work-stealing

Flip it around. Don't partition by thread up front — chop each pipeline's input into many
small chunks called **morsels** (≈10K–100K rows), and let a **fixed pool of worker
threads (one per core)** pull morsels from a shared queue:

```
pipeline input ──▶ [ morsel | morsel | morsel | morsel | morsel | … ]   (shared queue)
                       ▲          ▲          ▲
                    worker 1   worker 2   worker 3   ← grab next morsel when free,
                       │          │          │         push it through the whole pipeline
                       └──────────┴──────────┴──▶ shared sink (e.g. one hash table)
```

Each worker takes a morsel, pushes it through the **entire pipeline** to the sink, then
grabs the next morsel. Key consequences:

- **Self-balancing (work-stealing).** Fast threads simply process more morsels; there's
  no fixed partition to straggle on. Load balances automatically at runtime.
- **Dynamic parallelism.** The degree of parallelism is decided by the **scheduler at
  runtime**, not the optimizer at plan time — it can scale up/down and share cores across
  queries.
- **NUMA-aware.** A morsel is preferentially handed to a worker on the NUMA node that
  already holds its data (the paper's headline: "a NUMA-aware query evaluation
  framework").
- **Shared sink, concurrent.** All morsels of a pipeline feed *one* shared sink (e.g. a
  single concurrent hash table for the build side), not per-thread copies that must be
  merged.
- **Pipeline phases synchronize.** A pipeline breaker is a barrier: every morsel of the
  build pipeline must finish before the probe pipeline starts. (Those are the FULL /
  PARTIAL / PIPELINE barriers from [`push-vs-pull-execution.md`](push-vs-pull-execution.md).)

| | Exchange-operator (plan-driven) | Morsel-driven (data-driven) |
|---|---|---|
| Degree of parallelism | fixed at plan time | dynamic at runtime |
| Load balancing | partition-based → skew stragglers | work-stealing → self-balancing |
| Threads | one per partition/operator | fixed pool, dispatched morsels |
| NUMA | oblivious | aware |
| Sink | per-thread, merged later | one shared, concurrent |

A **morsel is to parallelism what a vector is to execution**: the small unit of work the
scheduler hands around. (They're related but distinct: a vector is ~2048 rows tuned to
cache; a morsel is a larger chunk, ~10K–100K rows, tuned to be a worthwhile unit of
*scheduling* — a worker processes a morsel as a series of vectors.)

DuckDB uses morsel-driven parallelism. It's the standard for modern analytical engines.

## Sirius: morsel-driven, with GPUs as the "cores"

Sirius isn't a line-by-line implementation of the Leis paper, but its task-based engine
*is* morsel-driven parallelism adapted to GPUs — same philosophy, GPU-shaped pieces:

| Morsel-driven concept | Sirius equivalent |
|---|---|
| Morsel (unit of work) | a **data batch** / scan-task chunk pushed through a pipeline |
| Worker pool (one per core) | **GPU worker threads** per GPU (+ scan/downgrade pools), across *all* GPUs |
| Shared morsel queue / work-stealing | task queues drained by whichever worker is free |
| Runtime scheduler (not plan-time) | **`task_creator`** decides what to schedule from data availability + barriers |
| Pipeline-phase barriers | FULL / PARTIAL / PIPELINE port barriers |
| NUMA-awareness | per-NUMA `sirius_ioctx`, NUMA-local host memory (`sirius_context`) — *plus* multi-GPU placement |

So Sirius generalizes the model: the "cores" are GPU streams/workers spread across
multiple GPUs; the "morsel" is a GPU-resident batch; the scheduler is the
`task_scheduler` + `task_creator`; and the readiness/barrier logic you saw in
[`file-maps/op/sirius_physical_operator.md`](file-maps/op/sirius_physical_operator.md)
(`get_next_task_hint`) is how it decides a morsel/task is ready to run. This is exactly
why the inter-pipeline handoff is *push-into-a-port-then-scheduled* rather than pull — a
work-stealing scheduler needs a buffer to steal from. The onboarding plan calls the Leis
paper "the scheduler's design philosophy" for precisely this reason.

## See also

- [`push-vs-pull-execution.md`](push-vs-pull-execution.md) — the push-into-ports +
  scheduler substrate morsel-driven scheduling runs on.
- [`vectorized-execution.md`](vectorized-execution.md) — vectors vs. morsels (execution
  unit vs. scheduling unit).
- [`weeks/week2-concepts.md`](weeks/week2-concepts.md) — Step 8 (task
  creation) and the Week 4–6 subsystems this sets up.
- *Paper:* Leis et al., "Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation
  Framework for the Many-Core Age" (2014).
