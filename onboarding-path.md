# Sirius Onboarding Path

A ~6–7 week plan to go from "I know modern C++ but not databases" to making
non-trivial contributions to Sirius. Calibrated to ~34 hrs/week (6 hrs/weekday +
2 hrs/weekend day).

**Milestones:**
- First small contribution: end of Week 2–3
- Non-trivial contribution: Week 6–7

**Working rule:** Never read a Sirius doc cold. For each doc below, first spend
20–30 min on the matching database concept (see [Resources](#resources)) so the
doc lands on prepared ground. Keep [`docs/glossary.md`](docs/glossary.md)
open as your term anchor, and use an LLM for *targeted* questions once you can point
at specific files and line numbers.

> Note: source/doc paths in this file (e.g. `src/op/...`, `docs/super-sirius/...`)
> are relative to the **Sirius repo root**. Links to other study notes are relative to
> this `sirius-onboarding-study` root.

**Effort tags:** some files carry a depth tag — *line count is a poor proxy for
effort, so these signal how hard to lean in:*
- *skim* — get the shape, don't sweat details
- *read* — understand the logic
- *study* — slow, line-by-line; the hard ones

Untagged files are self-evidently small; just read them.

---

## Week 1 — Build it, run it, learn the vocabulary

Goal: working build, a query you ran yourself, and the core DB execution vocabulary.

- [ ] **Day 1 — Get it building.** Its own task; budget the day.
  - [ ] `pixi shell`
  - [ ] `git submodule update --init --recursive`
  - [ ] Build per `CLAUDE.md`, then `make test`
  - [ ] Run one TPC-H query via `CALL gpu_execution('SELECT ...')`
- [x] **Day 2 — Big picture.** Read *"Architecture of a Database System"* — **Section 4
  only** (Relational Query Processor). Then `docs/README.md` and
  `docs/super-sirius/README.md`.
- [x] **Day 3 — Query execution model.** CMU 15-445 *Query Execution / Processing
  Models* lectures (push vs. pull, vectorized). Then
  `docs/super-sirius/architecture-overview.md`.
- [x] **Day 4 — Relational operators.** CMU 15-445 *Joins* and *Aggregations/Sorting*
  lectures. Then `docs/glossary.md` (all of it) and `docs/super-sirius/operators.md`.
- [x] **Day 5 — DuckDB, the host.** Read the DuckDB paper + skim DuckDB extension
  docs. Then read `docs/super-sirius/execution-flow.md` as a *map* (walk it in code
  next week).
- [ ] **Weekend (4h).** Re-read architecture + execution-flow docs. Write a one-page
  glossary in your own words; have an LLM quiz you.

**Checkpoint:** Can explain physical plan, operator, pipeline, and hash join in
plain language, and roughly how SQL becomes GPU work.

---

## Week 2 — Trace one query end-to-end in the code

Goal: follow a real query through actual source. Highest-leverage week.

Target query: `SELECT l_returnflag, SUM(l_quantity) FROM lineitem GROUP BY l_returnflag`
(scan + aggregate, no join).

- [ ] **Days 1–2 — Entry → plan** (keep `execution-flow.md` open). Two doorways into
      the engine, then the shared engine. Read the explicit doorway first (cleanest
      linear trace), then the transparent one (the production default); both converge
      on `sirius_interface`:
  - [x] `src/sirius_extension.cpp` — *skim*; Step 0 (extension load) + the explicit
        `gpu_execution` doorway (Step 1b). Large (~1,700 lines) but mostly
        registration/config boilerplate
  - [x] `src/transparent/sirius_optimizer_extension.cpp` + `src/transparent/physical_sirius_execution.cpp`
        — *read*; the **primary** doorway: plain SQL intercepted via optimizer hooks →
        `OnFinalizePrepare` → `PhysicalSiriusExecution`, which calls the same
        `sirius_execute_query`. Small (101 + 175 lines)
  - [x] `src/sirius_interface.cpp` — *read*; DuckDB-facing API, query lifecycle (where
        both doorways meet)
  - [x] `src/sirius_engine.cpp` — *read closely*; pipeline construction, the
        orchestration core
  - [x] `src/planner/sirius_physical_plan_generator.cpp` — *read*; logical→physical,
        + `docs/super-sirius/physical-plan-generation.md`
- [ ] **Day 3 — Ownership & lifecycle.** `src/include/sirius_context.hpp` + the
  "Ownership Hierarchy" section of the architecture doc.
- [ ] **Days 4–5 — Operators in code.**
  - [ ] `src/op/sirius_physical_operator.cpp` — *read*; the base interface every
        operator implements
  - [ ] `src/op/sirius_physical_limit.cpp` (warm-up; tiny)
  - [ ] `src/op/sirius_physical_duckdb_scan.cpp` (tiny)
  - [ ] `src/op/sirius_physical_ungrouped_aggregate.cpp` — *read*; closes the loop
  - [ ] Use `/module-context` so relevant cuDF API docs load on cuDF calls.
- [ ] **Weekend (4h).** Set `SIRIUS_LOG_LEVEL=debug`, run the query, then use
  `tools/parse_pipeline_log.py` to match per-operator row counts to the operators
  you read.

**Checkpoint:** Can point at the file + function for each stage of a query's life.
**Ready for a small contribution.**

---

## Week 3 — Operators, expressions, and your first contribution

Goal: depth on GPU computation in operators, and land a small PR.

- [ ] **Days 1–2 — Expressions on GPU.** `docs/super-sirius/expression-executor.md`
  + `src/expression_executor/`. Load `cudf/modules/ast.md` and `unary_binary.md`
  via `/module-context`.
- [ ] **Day 3 — A medium operator.** *read closely* — `src/op/sirius_physical_top_n.cpp`
  or `sirius_physical_partition.cpp`, plus its test in `test/cpp/operator/`.
- [ ] **Days 4–5 — Ship something small.** Add an SQL logic test under `test/sql/`,
  fix a small issue, or extend an operator's supported cases.
  - [ ] Read `CONTRIBUTING.md`
  - [ ] `pre-commit run -a`
  - [ ] Open a PR
- [ ] **Weekend (4h).** `docs/super-sirius/configuration.md` (tuning knobs you'll
  need constantly).

**Checkpoint:** First contribution submitted. Understand operator + expression layer.

---

## Weeks 4–5 — Concurrency, scheduling, and the scan subsystem

The genuinely hard core. Two weeks. Bugs here are races and leaks — don't rush.

- [ ] **DB theory slice (~3–4h across both weeks).** Read *"Morsel-Driven
  Parallelism"* (Leis et al., 2014) — the basis for task-based parallel execution.
- [ ] **Week 4 — Pipeline execution & task creation.**
  - [ ] `docs/super-sirius/pipeline-execution.md`
  - [ ] `src/include/pipeline/task_scheduler.hpp`, `gpu_pipeline_executor.hpp` (+ `.cpp`)
        — *study*; the concurrency core, where races live
  - [ ] `docs/super-sirius/task-creator.md` + `src/creator/` — *read closely*
  - [ ] Focus: the 5 thread pools, task lifecycle, completion signaling, hint chain.
- [ ] **Week 5 — Scan & data flow.**
  - [ ] `docs/super-sirius/scan.md` (largest doc), then the scan subsystem: *read*
        `src/include/op/scan/duckdb_scan_executor.hpp` and the executor logic in
        `src/op/scan/`, but *skim* the GPU parquet decoders (`src/cuda/scan/`,
        ~5–6k lines — get the shape, don't study them unless a task takes you there)
  - [ ] `docs/super-sirius/data-management.md` — *read*; data batches, repositories,
        ports, barriers (the wiring between pipelines)
  - [ ] `src/op/sirius_physical_hash_join.cpp` — *study*; the hard one, dense logic
        (1,224 lines, the most complex operator). Pair with CMU's hash join lecture.
- [ ] **Weekends (4h each).** Re-trace a query *with a join*
  (`SELECT ... FROM lineitem JOIN orders ...`) end-to-end, including multi-pipeline
  scheduling and the join build/probe split.

**Checkpoint:** Understand how work is scheduled across threads/GPUs and how data
flows between pipelines without races.

---

## Weeks 6–7 — Memory, multi-GPU, and a non-trivial contribution

- [ ] **Week 6 — Memory & fallback.**
  - [ ] `docs/super-sirius/memory-management.md` + `src/memory/`,
        `src/include/memory/sirius_memory_reservation_manager.hpp` — *read closely*;
        reservations and tiering are easy to get subtly wrong
  - [ ] `src/downgrade/` — *read*; GPU→host→disk spilling via cuCascade
  - [ ] `src/fallback.cpp` — *read* — + `docs/super-sirius/dynamic-filters.md`,
        `docs/super-sirius/optimizations.md`
  - [ ] Skim `docs/super-sirius/multi-gpu-architecture.md` (read deeply only if your
        work touches it)
  - [ ] GPU slice: skim libcudf C++ dev guide + relevant
        `.claude/skills/module-discover/docs/cudf/` modules.
- [ ] **Week 7 — Non-trivial contribution.** Add/extend an operator, improve a
  scheduling/memory path, or add a fallback case.
  - [ ] `/module-context` before implementing
  - [ ] Follow "Adding New Operators" in `CLAUDE.md`
  - [ ] Write both `test/cpp/` and `test/sql/` tests
  - [ ] `/code-review` before pushing

**Checkpoint:** Can make non-trivial contributions and reason about correctness
across the concurrency and memory layers.

---

## Resources

Use a thin slice — don't over-invest.

**Databases (priority order):**
1. **"Architecture of a Database System,"** Hellerstein/Stonebraker/Hamilton (2007)
   — Section 4 mandatory; rest optional. Best single overview.
2. **CMU 15-445 Intro to Database Systems** (Andy Pavlo, free on YouTube) — watch
   only: Processing Models / Query Execution, Joins, Aggregation & Sorting, Query
   Planning & Optimization, Vectorized Execution (~5–6 lectures, not the whole course).
3. **DuckDB paper** ("DuckDB: an Embeddable Analytical Database") + DuckDB extension
   docs — your host system.
4. **"Morsel-Driven Parallelism"** (Leis et al., 2014) — for Weeks 4–5; the
   scheduler's design philosophy.
5. *Optional:* **"MonetDB/X100: Hyper-Pipelining Query Execution"** — origin of
   vectorized/columnar execution.

**GPU/cuDF (just-in-time, not up front):**
- RAPIDS **libcudf C++ developer guide** + the pre-extracted docs in
  `.claude/skills/module-discover/docs/{cudf,rmm,cucascade}/`. Let `/module-context`
  pull the right ones per task. You do **not** need to write CUDA kernels to
  contribute — `src/cuda/` is advanced and can wait.

**Skip for now:** OLTP/storage-engine material (B-trees, MVCC, WAL, transactions).
Alex Petrov's *Database Internals* is great but mostly about a transactional,
disk-storage world different from this analytical GPU engine.

**Ignore in the source tree:** `src/legacy/` (~23k lines, the old `gpu_processing`
path). All new development targets Super Sirius (`namespace sirius`, `src/op/`).
