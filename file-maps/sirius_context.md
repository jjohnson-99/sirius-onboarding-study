# `sirius_context.hpp` → Ownership & Lifecycle Map

Companion for Week 2, **Day 3** of [`onboarding-path.md`](onboarding-path.md),
tagged **read closely**. Read `src/include/sirius_context.hpp` (the header is the
point; `src/sirius_context.cpp` holds the implementations) alongside the **"Ownership
Hierarchy"** section of `docs/super-sirius/architecture-overview.md`.

> Source/doc paths are relative to the Sirius repo root. Line numbers
> accurate as of 2026-06-10; re-grep if the file moved.

## Where this sits

Every file you've read so far reaches into `SiriusContext` through the same keyed
lookup:

```cpp
context.registered_state->Get<duckdb::SiriusContext>("sirius_state")
```

You saw it in `sirius_extension.cpp` (pin_table, query label), `sirius_engine.cpp`
(`create_query`, `get_task_scheduler`), and you'll see it in every executor. **This
file is the thing on the other end of that lookup** — the single object that *owns*
every Sirius subsystem for a DuckDB connection. Day 3 is about ownership and
lifetime, not query flow, which is why the plan reads it on its own.

## The big idea: one owner, attached to DuckDB's connection

`SiriusContext` is a **`duckdb::ClientContextState`** subclass (line 75). DuckDB lets
an extension attach per-connection state and gives it lifecycle callbacks;
`SiriusContext` uses that hook to own the whole GPU runtime and to intercept queries.
Two roles in one class:

1. **Owner** of all subsystems (the architecture doc's "Ownership Hierarchy").
2. **Interceptor** of DuckDB's query lifecycle (the hooks that drive *transparent*
   execution — the doc's Step 1).

## Role 1 — Ownership hierarchy (the private members, 325–443)

These `unique_ptr` members *are* the ownership tree from the architecture doc. The
comments in this region are unusually important: they document **destruction order**,
which is load-bearing for correctness (GPU resources must be torn down under a live
CUDA context, in dependency order). The subsystems:

| Member | What it owns | You'll read it in |
|---|---|---|
| `config_` | `sirius_config` (thread counts, memory sizes, `operator_params`) | `configuration.md` (Wk 3) |
| `memory_manager_` | `sirius_memory_reservation_manager` (GPU/host/disk tiers) | Wk 6 |
| `data_repository_manager_` | the registry of `shared_data_repository` between pipelines | Wk 5 |
| `task_scheduler_` | top-level executor (owns GPU + scan executors) | Wk 4 |
| `downgrade_executors_` | per-space GPU→host spilling monitors | Wk 6 |
| `task_creator_` | creates scan/GPU tasks from data availability | Wk 4 |
| `scan_manager_` | parquet split providers / scan caching | Wk 5 |
| `numa_ioctxs_` / `gpu_ioctxs_` / `s3_ioctx_` | per-NUMA + per-GPU IO contexts | Wk 5–6 |
| `query_` | the current `sirius::planner::query` (pipeline set) | Wk 4 |

You don't need to understand each subsystem now — the point of Day 3 is to see that
**they all hang off this one object**, with public accessors (`get_memory_manager()`,
`get_task_scheduler()`, `get_task_creator()`, `get_scan_manager()`,
`get_data_repository_manager()`, …) that the rest of the codebase uses. Treat this as
the map you'll return to whenever you wonder "who owns X / how do I reach X."

## Role 2 — Query lifecycle & transparent execution (public, 92–116)

| Method | Line | Role |
|---|---|---|
| `void QueryBegin(ClientContext&)` | 94 | Per-query setup hook (resets operator IDs / task-creator state). |
| `void QueryEnd(...)` (3 overloads) | 97–106 | Per-query teardown (clears data repositories). |
| `bool CanRequestRebind()` | 109 | Returns `true` so DuckDB will call `OnFinalizePrepare`. |
| `RebindQueryInfo OnFinalizePrepare(ClientContext&, PreparedStatementData&, PreparedStatementMode)` | 113 | **The transparent path's Step 1.** After DuckDB builds its CPU physical plan, this calls the **plan generator's `create_plan()`** on the captured logical plan and, if it succeeds, swaps DuckDB's plan for a Sirius one. If `create_plan()` throws → CPU plan stays → silent fallback. This is the transparent-mode analogue of `GPUExecutionBind`. |

So both entry doorways converge here for *plan generation*: the explicit
`gpu_execution` path calls the plan generator from `sirius_extension.cpp`; the
transparent path calls it from `OnFinalizePrepare` in this file. (See
[`sirius_extension.md`](sirius_extension.md) Step 0/1b.)

Supporting transparent-mode state: `set/take_captured_logical_plan` (292/296) —
the optimizer hook stashes the optimized logical plan here for `OnFinalizePrepare` to
consume; the `transparent_execution_stats` counters track rebinds/fallbacks.

## The one mechanism to really understand: `InternalQueryGuard` (133–157)

Because the **same** `SiriusContext` is registered on *every* connection to the
database (see `SiriusContextExtensionCallback::OnConnectionOpened`, 454), any code
that opens a *second* internal DuckDB connection (e.g. Iceberg metadata reads — recall
`prefetch_iceberg_delete_data` in [`sirius_engine.md`](sirius_engine.md)) would fire a
nested `QueryBegin`/`QueryEnd` and **corrupt the outer query's state**.
`InternalQueryGuard` is a RAII depth-counter that suppresses those callbacks for the
duration of the internal connection. When you see `InternalQueryGuard guard(*ctx)` in
the codebase, that's what it's protecting against. Also note
`query_lifecycle_mutex_` (336): the shared runtime serializes one query lifecycle at a
time across connections.

## `SiriusContextExtensionCallback` (448–475)

A small `ExtensionCallback` that registers/unregisters a `SiriusContext` as
connections open/close, and reads the config file. This is the glue that makes the
`"sirius_state"` lookup succeed everywhere; it's installed in `LoadInternal`
(`sirius_extension.cpp`).

## Types fundamental to *this* file

- **`duckdb::ClientContextState`** *(DuckDB base)* — the extension hook for
  per-connection state + lifecycle callbacks. **Think:** "DuckDB's officially
  sanctioned place to bolt Sirius onto a connection."
- **`SiriusContext`** *(this file)* — the owner-of-everything + query interceptor.
  **Think:** the Sirius runtime, scoped to a connection; the target of every
  `"sirius_state"` lookup.
- The subsystem types (`task_scheduler`, `sirius_memory_reservation_manager`,
  `task_creator`, `sirius_scan_manager`, `downgrade_executor`,
  `shared_data_repository_manager`) — *named here, read later.* Don't dive in now; just
  note this is their home and their accessor names.
- **`PreparedStatementData` / `RebindQueryInfo`** *(DuckDB)* — the in/out of
  `OnFinalizePrepare`; the mechanism by which Sirius substitutes its plan for DuckDB's.

## Takeaway

`SiriusContext` is the **root of the ownership tree** and the **transparent-execution
interceptor**. Hold two things from Day 3: (1) the accessor names for each subsystem
(your map for Weeks 4–6), and (2) the `"sirius_state"` lookup + `InternalQueryGuard`
pattern. Next, Days 4–5 drop from this top-level wiring down into individual operators
— starting with the base interface every operator implements.
