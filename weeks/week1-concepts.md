# Week 1 Concepts — Physical Plan, Operator, Pipeline, Hash Join

The Week 1 checkpoint in [`onboarding-path.md`](../onboarding-path.md):

> Can explain physical plan, operator, pipeline, and hash join in plain language,
> and roughly how SQL becomes GPU work.

This is that explanation, in plain language, loosely cross-referenced with the three
Week 1 readings:

- **Ch. 4** = *"Architecture of a Database System"* (Hellerstein/Stonebraker/Hamilton),
  Section 4, "Relational Query Processor."
- **CMU 15-445** = the Fall 2025 Intro to Database Systems notes/lectures (Query
  Execution, Joins).
- **DuckDB paper** = *"DuckDB: an Embeddable Analytical Database."*

---

## The one-sentence version

SQL text is parsed and optimized into a tree of relational **operators** (the
**physical plan**); the engine runs that tree by streaming **batches** of rows
through chains of operators called **pipelines**; Sirius intercepts DuckDB's physical
plan and re-implements those operators on the GPU using cuDF.

---

## Physical plan

A **physical plan** is a tree of concrete algorithms that, executed bottom-up,
produces the query's result. It's the *how*, as opposed to the **logical plan**'s
*what*.

- **Ch. 4** frames this as the query processor's job: parse → rewrite → **optimize**
  (logical algebra → physical operators with chosen algorithms and access paths) →
  execute. The optimizer is where "join these two tables" becomes "**hash** join,
  build on the right, probe on the left."
- **CMU 15-445** draws the same logical-vs-physical line: a logical operator (e.g.
  `⋈`, a join) maps to one of several physical operators (hash join, sort-merge join,
  nested-loop join). The optimizer picks based on cost and cardinality.
- **DuckDB paper** describes DuckDB doing exactly this internally, then handing a
  finished physical plan to its vectorized execution engine.

In Sirius, the plan you care about is the one DuckDB *already optimized*. Sirius does
not re-plan from SQL — it walks DuckDB's logical plan and builds a **parallel**
physical tree of its own GPU operators in
`src/planner/sirius_physical_plan_generator.cpp`. So "physical plan" has two layers
here: DuckDB's, and the Sirius mirror of it.

## Operator

An **operator** is one node in that tree — one relational transformation. Scan,
filter, projection, aggregate, join, sort, limit. Each consumes rows from its
children and produces rows for its parent.

- **Ch. 4 / CMU 15-445**: the classic interface is the **iterator (Volcano) model** —
  every operator exposes `open()` / `next()` / `close()`, and `next()` pulls one
  tuple from below. Simple, but one-tuple-at-a-time is slow.
- **CMU 15-445** then introduces the fix used by every modern analytical engine:
  **vectorized / batch** execution — `next()` returns a *batch* of rows (a column
  chunk), not one tuple, so per-call overhead is amortized and the inner loops are
  tight and cache/SIMD-friendly.
- **DuckDB paper**: DuckDB is **vectorized** and **push-based** — operators push
  batches ("DataChunks") downstream rather than parents pulling one tuple at a time.

In Sirius, every operator is a `sirius_physical_operator` subclass in `src/op/`
(base interface in `src/op/sirius_physical_operator.cpp`). The "batch" is a
`cudf::table` living in GPU memory, and an operator's `execute()` is essentially a
call into cuDF. So Sirius takes the vectorized idea to its limit: a "vector" is a
whole GPU column, and the inner loop is a CUDA kernel over thousands of rows at once.

## Pipeline

A **pipeline** is a maximal chain of operators that can pass a batch straight through
without anyone having to stop and collect *all* the rows first.

The key distinction (from **CMU 15-445**):

- **Pipeline-breaker** (a.k.a. blocking) operators must see their *entire* input
  before producing output — a hash join's build side, a sort, a full aggregation.
  They force a boundary.
- **Streaming** operators (filter, projection) pass each batch through immediately.

So a plan is cut into pipelines *at* the pipeline-breakers. Within a pipeline a batch
flows source → ... → sink without materializing the whole intermediate; at a breaker,
the **sink** accumulates state (e.g. builds a hash table), and a *new* pipeline starts
downstream.

- **Ch. 4** discusses this as the execution engine streaming data through operators
  and materializing only where an algorithm requires it.
- **DuckDB paper**: DuckDB explicitly organizes execution into pipelines and runs
  them with **morsel-driven parallelism** (you'll read the Leis paper in Weeks 4–5) —
  many threads each grab a small chunk ("morsel") of the same pipeline.

Sirius makes pipelines a first-class, owned thing. `src/sirius_engine.cpp` builds a
`sirius_meta_pipeline` from the physical tree, **splits** it at breakers (joins,
aggregates, sorts, scans), and injects PARTITION / CONCAT / MERGE operators at the
boundaries. Each pipeline becomes GPU **tasks** scheduled across GPU worker threads
(and across multiple GPUs). This is why "pipeline" shows up everywhere in the Sirius
docs — it's the unit of scheduling, not just a conceptual chain.

## Hash join

A **hash join** answers "match rows from table A to table B on a key" without
comparing every pair. Two phases (**CMU 15-445**'s Joins lecture):

1. **Build** — read the smaller relation, hash each row on the join key, insert into
   an in-memory **hash table**. This side is a **pipeline-breaker**: you must finish
   building before you can probe.
2. **Probe** — stream the larger relation; for each row, hash its key and look it up
   in the table, emitting matches.

- **Ch. 4** presents hash join as the workhorse physical operator for equi-joins, and
  notes the build/probe asymmetry (build the small side) and the memory question (what
  if the hash table doesn't fit — partitioning / spilling).
- **DuckDB paper**: DuckDB implements a vectorized, parallel hash join — the build
  side fills a shared hash table, then probe-side pipelines run in parallel against
  it. The build is the pipeline boundary.

**How Sirius implements it** (`src/op/sirius_physical_hash_join.cpp`): the *shape* is
identical to the textbook and to DuckDB — a build side and a probe side, split into
two pipelines at the build boundary (`build_join_pipelines()` constructs a child
meta-pipeline whose sink is the join's build). The difference is the **inner
machinery**: instead of a CPU hash table probed one batch at a time, Sirius hands the
key columns to **cuDF's GPU join primitives** — `cudf::hash_join`,
`cudf::inner_join` / `left_join` / `full_outer_join`, and `cudf::filtered_join` for
semi/anti-style cases. cuDF builds the hash table *on the GPU* and probes it with a
massively parallel kernel. Sirius adds planning-time smarts on top: if it can *prove*
the build keys are unique by walking DuckDB's logical plan (e.g. a PRIMARY KEY), it
swaps in the faster `cudf::distinct_hash_join`. Pure-inequality joins that cuDF's
hash join can't express fall back to `sirius_physical_nested_loop_join`.

So the concept ("build a hash table on the small side, probe with the big side, break
the pipeline at the build") is the same one you read about; Sirius just executes both
phases as GPU kernels over whole columns.

---

## How SQL becomes GPU work (end to end)

Tying it together — the path a query like
`SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY l_returnflag` takes:

1. **SQL → logical plan → optimized physical plan** — *DuckDB* does this (its parser,
   optimizer, and physical planner; the Ch. 4 / DuckDB-paper pipeline). Sirius does
   **not** re-plan from text.
2. **Sirius intercepts the physical plan** — as a DuckDB **extension**
   (`src/sirius_extension.cpp`, `src/sirius_interface.cpp`), Sirius gets DuckDB's
   plan and walks it in `sirius_physical_plan_generator.cpp`, building a mirror tree
   of `sirius_physical_operator`s. Unsupported bits **fall back** to DuckDB's CPU
   engine (`src/fallback.cpp`).
3. **Plan → pipelines** — `sirius_engine.cpp` splits the operator tree into pipelines
   at the breakers (here: the scan and the GROUP BY aggregate) and wires data
   repositories between them.
4. **Pipelines → GPU tasks** — the scheduler turns pipelines into tasks; scan tasks
   pull `lineitem` from storage into GPU columns, GPU worker threads run each
   operator's `execute()` as cuDF/CUDA calls on a CUDA stream.
5. **Operators run on the GPU** — each operator *is* a cuDF call over a `cudf::table`:
   the aggregate becomes a `cudf::groupby`, a join would become `cudf::hash_join`,
   etc. Memory is managed in tiers (GPU → host → disk) and spilled under pressure.
6. **Result back to DuckDB** — the final pipeline materializes a result that DuckDB
   hands back to the client.

**The mental model to keep:** DuckDB decides *what* to compute and *how* (plan +
optimizer + the vectorized/push/pipeline ideas from your readings); Sirius re-runs the
*how* on the GPU by mapping each vectorized operator onto a cuDF primitive over GPU
columns, and falls back to DuckDB whenever the GPU can't handle a case. Everything in
Weeks 2+ is just watching this happen in the actual source.
