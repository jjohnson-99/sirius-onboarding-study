# Transparent-Doorway Map — `transparent/sirius_optimizer_extension.cpp` + `physical_sirius_execution.cpp`

Companion for Week 2, **Days 1–2** of [`onboarding-path.md`](../../onboarding-path.md).
Read this **right after** [`file-maps/sirius_extension.md`](../sirius_extension.md): that file
is the *explicit* doorway (`CALL gpu_execution(...)`); this is the **primary**
doorway — plain SQL, transparently intercepted. Read
`src/transparent/sirius_optimizer_extension.cpp` (101 lines) and
`src/transparent/physical_sirius_execution.cpp` (175 lines) with **Step 1** and
**Step 2** of `docs/super-sirius/execution-flow.md` open.

> Source/doc paths are relative to the Sirius repo root. Line
> numbers accurate as of 2026-06-11; re-grep if a file moved.

## Why this exists (and why it's the primary path)

With a Sirius config present, a user just writes plain SQL — no `CALL`. Sirius has to
intercept the query *inside DuckDB's own pipeline* and quietly route it to the GPU.
That interception is what these files do. It's the default; the explicit
`gpu_execution(...)` table function is "still supported but no longer primary" (the
"(Legacy)" tag on Step 1b — see the note in
[`file-maps/sirius_extension.md`](../sirius_extension.md)).

**The crucial payoff:** this doorway and the explicit one **converge on the exact
same engine entry point**. Both end up calling
`sirius_interface::sirius_execute_query(...)`. So everything you mapped from Step 3
onward ([`file-maps/sirius_interface.md`](../sirius_interface.md) → engine → operators) is
shared by both paths. Only the doorway differs.

## The transparent path has three pieces

| Piece | File | Map |
|---|---|---|
| 1. Optimizer hooks — capture the optimized logical plan | `transparent/sirius_optimizer_extension.cpp` | **this file** |
| 2. `OnFinalizePrepare` — swap DuckDB's physical plan for a Sirius one | `sirius_context.cpp` | [`file-maps/sirius_context.md`](../sirius_context.md) (Role 2) |
| 3. `PhysicalSiriusExecution` — the substituted operator that runs the GPU query | `transparent/physical_sirius_execution.cpp` | **this file** |

They run in that order, hooked into DuckDB's normal query lifecycle. The hooks are
*registered* back in `LoadInternal` (`sirius_extension.cpp`, lines 1645–1648) — which
is why Step 0 (extension load) is a prerequisite for this path.

## Call sequence

This map covers **two** source files (and one method in a third); each block below is
labeled with its file. `─▶` leaves these files; `· Step N` = `execution-flow.md` step:

```
sirius_optimizer_extension.cpp — install (during DuckDB's optimize → prepare):
  sirius_pre_optimizer_hook()        :40   disable incompatible optimizers              · Step 1
  sirius_optimizer_hook()            :74   ─▶ SiriusContext.set_captured_logical_plan  (stash only)
  ════ control returns to DuckDB; it finishes preparing the statement ════
  DuckDB: ClientContext::PrepareInternal     ◀─ calls the hook (NOT the optimizer hook above)
     └─▶ SiriusContext::OnFinalizePrepare()  (sirius_context.cpp:859)  [gated on CanRequestRebind]
                                           ─▶ create_plan → installs PhysicalSiriusExecution

physical_sirius_execution.cpp — run (DuckDB drives PhysicalSiriusExecution as a source op):
GetGlobalSourceState()                :68   creates the sirius_interface (once)          · Step 2
GetDataInternal()                     :86 ◀─ called repeatedly by DuckDB
├─ first call only ──▶ lazy build & run:
│  ├─ logical_plan_->Copy()  (or re-parse+optimize :132)  :119   rebuild a fresh plan
│  ├─ planner.create_plan(fresh_plan) :143  ─▶ plan generator
│  ├─ wrap → sirius_prepared_statement_data :146
│  └─ iface->sirius_execute_query(…)  :151  ─▶ sirius_interface               · Step 3 → 9
└─ state.result->Fetch()              :164   pull the next chunk → HAVE_MORE / FINISHED
```

The first block (`sirius_optimizer_extension.cpp`) is DuckDB's optimize/prepare phase
capturing the plan and swapping in the operator (via `OnFinalizePrepare` in
`sirius_context.cpp`); the second (`physical_sirius_execution.cpp`) is that operator being
pulled for data. The `iface->sirius_execute_query` handoff at `:151` is **the convergence
point** — the explicit `gpu_execution` path reaches the *same* call (see
[`file-maps/sirius_interface.md`](../sirius_interface.md) → Call sequence), so from
Step 3 on the two doorways are identical.

## Piece 1 — the optimizer hooks (Step 1)

DuckDB's `OptimizerExtension` lets you run code around its optimizer. Sirius
registers two functions:

| Function (signature) | Line | Role |
|---|---|---|
| `bool gpu_execution_enabled(const ClientContext&)` | 31 | **The on/off switch.** Reads the `gpu_execution` *boolean setting* (default `true`, registered in `InitialGPUConfigs`). If false, both hooks no-op and the query stays on DuckDB CPU. (Note: same name `gpu_execution` as the table function — here it's the SET toggle.) |
| `void sirius_pre_optimizer_hook(OptimizerExtensionInput&, unique_ptr<LogicalOperator>& plan)` | 40 | **Before** optimization: snapshots the connection's disabled-optimizer set, then disables passes Sirius can't yet execute — `IN_CLAUSE`, `COMPRESSED_MATERIALIZATION`, `STATISTICS_PROPAGATION`, `LATE_MATERIALIZATION` (each with a comment on *why*). |
| `void sirius_optimizer_hook(OptimizerExtensionInput&, unique_ptr<LogicalOperator>& plan)` | 74 | **After** optimization: restores the original disabled set (so CPU queries aren't affected), then **copies the optimized logical plan** and stashes it in `SiriusContext` via `set_captured_logical_plan`. On copy failure → silently skip GPU. |

Both hooks early-out unless `gpu_execution_enabled` **and** the `SiriusContext` is
initialized **and** it's not an internal query (`is_internal_query_active()` — the
`InternalQueryGuard` guard you met in [`file-maps/sirius_context.md`](../sirius_context.md)).

> The disable/restore is **per-query** and restored *unconditionally* right after
> optimization (with a `QueryEnd` backstop) — **not** only on CPU fallback. Note also
> that `disabled_optimizers` is database-wide `DBConfig` state, not per-connection.
> Worked through in
> [`reference/explainers/client-connections.md`](../../reference/explainers/client-connections.md).

This is the explicit path's `PrepareConnection` optimizer-disabling, relocated into a
DuckDB hook. Same idea, different trigger — exactly the symmetry noted in
[`file-maps/sirius_extension.md`](../sirius_extension.md).

## Piece 2 — `OnFinalizePrepare` (in `sirius_context.cpp`)

Not in these files, but it's the hinge: after DuckDB builds its CPU physical plan,
`SiriusContext::OnFinalizePrepare` takes the captured logical plan, calls the **plan
generator's `create_plan()`** (the single source of truth for GPU support), and — if
it succeeds — replaces DuckDB's physical plan with a `PhysicalSiriusExecution`. If
`create_plan()` throws (unsupported op) the CPU plan stays → **silent fallback**.
Full treatment in [`file-maps/sirius_context.md`](../sirius_context.md).

> **Important: the optimizer hook does *not* call `OnFinalizePrepare`.** This is
> capture-then-callback. `sirius_optimizer_hook` only *stashes* the plan
> (`set_captured_logical_plan`) and returns; **DuckDB itself** later calls
> `OnFinalizePrepare` during prepare-finalization (`ClientContext::PrepareInternal`),
> but only on registered states whose `CanRequestRebind()` returns `true` — which
> `SiriusContext` overrides to do. So the two pieces are connected *through DuckDB*, not by
> a direct call. See [`file-maps/sirius_context.md`](../sirius_context.md) →
> "Who calls `OnFinalizePrepare`?" for the registration + gate detail.

## Piece 3 — `PhysicalSiriusExecution` (Step 2)

A DuckDB `PhysicalOperator` subclass that *is* the Sirius query, slotted into DuckDB's
plan. DuckDB drives it like any source operator:

| Function (signature) | Line | Role |
|---|---|---|
| ctor `PhysicalSiriusExecution(PhysicalPlan&, unique_ptr<LogicalOperator> logical_plan, string query_sql, types, names, card)` | 50 | Holds the logical plan + original SQL so it can (re)build a fresh Sirius plan per execution. |
| `GetGlobalSourceState(ClientContext&)` | 68 | Creates the `sirius_interface` (and takes any pending query label). The `SiriusGlobalSourceState` (37) owns the interface + the materialized result. |
| `GetDataInternal(ExecutionContext&, DataChunk&, OperatorSourceInput&)` | 86 | **The convergence point.** Lazily on first call: rebuild a fresh Sirius physical plan (via `logical_plan_->Copy()`, or re-parse+re-optimize the SQL if the plan isn't copyable), wrap it in a `sirius_prepared_statement_data`, and call **`state.iface->sirius_execute_query(...)`** (line 151). Then `Fetch()` chunks out to DuckDB until drained. |

So `PhysicalSiriusExecution::GetDataInternal` is to the transparent path what
`GPUExecutionFunction` is to the explicit path: the spot that calls
`sirius_execute_query`. From there, **Step 3 onward is identical** between the two.

## The two doorways side by side

```
            ┌─ register hooks + PhysicalSiriusExecution swap (Step 1) ─┐
plain SQL ─▶ optimizer hooks ─▶ OnFinalizePrepare ─▶ PhysicalSiriusExecution.GetDataInternal ─┐
                                                                                               ├─▶ sirius_interface
CALL gpu_execution(...) ─▶ GPUExecutionBind ─▶ GPUExecutionFunction ───────────────────────────┘    ::sirius_execute_query
                                                                                                      (Step 3 → … shared)
```

## Types fundamental to *this* file

Recurring DuckDB types (`ClientContext`, `LogicalOperator`, `DataChunk`,
`PreparedStatementData`) → [`reference/duckdb-types-glossary.md`](../../reference/duckdb-types-glossary.md).
Specific here:

- **`OptimizerExtension` / `OptimizerExtensionInput`** *(DuckDB)* — the hook mechanism
  DuckDB exposes for running code around its optimizer. **Think:** "DuckDB's
  sanctioned seam for intercepting a query mid-optimization."
- **`PhysicalOperator` + `GlobalSourceState` + `SourceResultType`** *(DuckDB)* — the
  source-operator interface `PhysicalSiriusExecution` implements so DuckDB can pull
  result chunks from it. **Think:** "a DuckDB operator whose `GetData` happens to run a
  whole GPU query."
- **`sirius_interface` / `sirius_prepared_statement_data`** *(Sirius)* — the shared
  engine entry; detail in [`file-maps/sirius_interface.md`](../sirius_interface.md). **Think:**
  the door both paths walk through.

## Takeaway

This is the production doorway, but it's *thin*: it captures a plan, swaps in an
operator, and that operator calls `sirius_execute_query`. Because both doorways
converge there, the rest of Week 2 ([`file-maps/sirius_interface.md`](../sirius_interface.md)
onward) covers both. Synthesis in
[`weeks/week2-concepts.md`](../../weeks/week2-concepts.md), Stage 1.
