# Push vs. Pull Execution

Push vs. pull is a question about **who's in charge of moving data** through the
operator tree — and the answer reshapes the whole engine architecture. It's orthogonal
to [`vectorized-execution.md`](vectorized-execution.md) (that's about *batch size*; this
is about *direction*), and it's the single best lens for understanding why Sirius has
ports, data repositories, and a task creator at all.

> **Prime around:** the *concept* in Week 1–2 (pipelines); most useful **entering Week
> 4**, where the ports/barriers/task-creator it explains are the files you read.

## The two directions

Every query is a tree of operators; data has to flow from the leaves (scans) up to the
root (the result). There are two ways to drive that flow.

**Pull (demand-driven / "Volcano").** The *consumer* asks. The root calls `next()` on
its child, which calls `next()` on its child, down to the scan. **Control flows
top-down; data flows bottom-up as return values.** The parent is in charge — "give me
the next batch."

```
result.next() ─▶ aggregate.next() ─▶ filter.next() ─▶ scan.next()
                                                          │  returns a batch
                            ◀── batch bubbles back up ────┘
```

**Push (data-driven).** The *producer* drives. The scan reads a batch and **pushes** it
into its parent's consume/`sink` method, which processes and pushes to *its* parent, up
to the root. **Control and data both flow bottom-up.** The child is in charge — "here's
the next batch, deal with it."

```
scan ──push──▶ filter ──push──▶ aggregate ──push──▶ result
(produce a batch, hand it upward; each operator reacts)
```

The everyday analogy: **pull = a lazy iterator / generator** (you call `next()` when you
want more); **push = a reactive callback / stream** ("don't call me, I'll call you").

## Trade-offs

| | Pull (demand-driven) | Push (data-driven) |
|---|---|---|
| Who drives | consumer (top) | producer (bottom) |
| Easy to implement | yes — recursive `next()` | needs a sink/consume interface |
| `LIMIT` / early stop | trivial — just stop pulling | trickier — must signal producer to stop |
| Backpressure | automatic (consumer paces) | must be engineered |
| Multiple consumers / DAGs | awkward — who pulls? | natural — producer fans out |
| Parallelism / scheduling | harder (control is inverted) | natural — produce a batch, hand it to a scheduler |

The short version: **pull is simpler and great for single-output trees and early
termination; push is better for pipelines, fan-out (CTEs, shared scans), and
parallelism.** That last point is why analytics engines lean push.

## The hybrid reality: pipelines

Modern engines don't pick one globally — they're **push-based within a pipeline**, with
the tree cut at **pipeline breakers** (blocking operators like sorts and hash-join
builds). A pipeline is a chain you push one batch through from source to sink without
stopping; at a breaker the sink *buffers* its whole input (builds a hash table,
accumulates an aggregate), and a **new** pipeline starts downstream. This is exactly the
pipeline concept from [`../../weeks/week1-concepts.md`](../../weeks/week1-concepts.md) —
push-vs-pull is the mechanism underneath it.

DuckDB: push-based vectorized pipelines with morsel-driven parallelism — each thread
grabs a chunk and pushes it through the pipeline to the sink.

## Sirius: push within a pipeline, scheduled handoff between them

This is where the concept pays off, because Sirius's whole runtime shape falls out of
it. Two regimes:

**Within a pipeline — push.** A task runs each operator's `execute()` over a batch
(push-through), then `publish_output()` calls the last operator's `sink()`, whose base
implementation **pushes** each output batch to its downstream ports via
`push_data_batch` (see
[`../../file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md)).
Straight producer-driven push, just like DuckDB.

**Between pipelines — push into a repository, then *scheduled* pull.** A sink doesn't
call the next operator directly; it pushes batches into a **`shared_data_repository`**
(a *port*). The downstream pipeline doesn't pull tuples on demand either. Instead the
**`task_creator`** polls each operator's `get_next_task_hint()` to ask "does this port
have enough data, given its barrier?" and, when ready, schedules a task that **pops**
batches from the port (`get_next_task_input_data`). So the inter-pipeline handoff is:

```
upstream sink ──push──▶ [shared_data_repository / port] ◀──poll readiness── task_creator
                                     │                                          │
                                     └────────── pop batches ◀── schedules ─────┘
                                                              (a consumer task)
```

This **decoupling** — producers push into buffers, a scheduler creates consumer tasks
when *barrier* conditions are met — is precisely what lets Sirius run many threads
across many GPUs and spill memory under pressure. A pure pull model couldn't express
"wait for the entire build side (FULL barrier) before the join runs, but stream
freely across a PIPELINE barrier" — that readiness logic *is* `get_next_task_hint`, and
it only makes sense in a push-into-ports world. (See Step 8, task creation, in
[`../../weeks/week2-concepts.md`](../../weeks/week2-concepts.md), and the
morsel-driven-parallelism paper in Weeks 4–5.)

So the one-line answer for Sirius: **push through a pipeline; between pipelines, push
into a port and let the task creator schedule the consumer when its barrier is
satisfied.** Ports, barriers, repositories, and the task creator are all machinery this
choice requires.

## See also

- [`vectorized-execution.md`](vectorized-execution.md) — the orthogonal axis (batch
  granularity); Sirius is both push-based *and* vectorized.
- [`../../weeks/week1-concepts.md`](../../weeks/week1-concepts.md) — pipelines and
  pipeline breakers.
- [`../../file-maps/op/sirius_physical_operator.md`](../../file-maps/op/sirius_physical_operator.md)
  — `sink()`/`push_data_batch`, ports/barriers, and `get_next_task_hint()` in code.
