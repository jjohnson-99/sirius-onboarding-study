# `sirius_extension.cpp` → Execution-Flow Map

Companion for Week 2, Days 1–2 of [`onboarding-path.md`](../onboarding-path.md): read
`src/sirius_extension.cpp` with `docs/super-sirius/execution-flow.md` open, and use
this to place each function against the doc's numbered steps.

> Source/doc paths (`src/...`, `docs/...`) are relative to the **Sirius repo root**;
> links to other study notes are relative to this file (normal Markdown relative paths). Line
> numbers were accurate as of the read on 2026-06-10 — if the file has changed since,
> re-confirm with a quick grep for the function name.

## The key idea first

`sirius_extension.cpp` is the **entry/registration layer** plus the explicit
`CALL gpu_execution(...)` path (the doc's **Step 1b**). It does **not** implement
Steps 2–9 — those live in other files. It *wires up* Step 1 (the transparent path)
by registering hooks, but the hook bodies live in `src/transparent/`.

So most of the doc maps *out* of this file. This is the "doorway, not the engine"
file: both entry paths converge on `sirius_interface.cpp` (Step 3), which is your
next file.

> **Two different "legacy"s — don't conflate them.** The execution-flow doc labels
> Step 1b (the explicit `gpu_execution` path) "Legacy," but that only means the
> `CALL gpu_execution('...')` *invocation style* is no longer the default — it still
> runs the **current Super Sirius engine**, and the `gpu_execution` table function is
> registered **unconditionally** (line 1192, *outside* any `#ifdef`). The *truly*
> legacy code — the old `gpu_processing` engine — is everything gated behind
> `#ifdef SIRIUS_ENABLE_LEGACY` (e.g. `gpu_processing`, `gpu_buffer_init`), compiled
> out by default. So: this file is fully live. We read the explicit path **first**
> because it's the cleanest linear trace of "SQL → engine"; the **primary** doorway
> (plain SQL, transparently intercepted) is
> [`transparent/sirius_optimizer_extension.md`](transparent/sirius_optimizer_extension.md),
> read right after this. Both converge on `sirius_interface`.

## Call sequence

Two unrelated flows live here — a one-time `load` and the per-call `gpu_execution` path.
`─▶` leaves this file; `· Step N` = `execution-flow.md` step:

```
load — Step 0 (once, at LOAD):
DUCKDB_CPP_EXTENSION_ENTRY(sirius, loader)     :1674
└─ LoadInternal(loader)                        :1623   the bootstrap — registers everything:
   ├─ GetCallbackManager().Register(…)         :1631   SiriusContextExtensionCallback (per-conn state)
   ├─ InitialGPUConfigs(config)                :1452   SET options (AddExtensionOption ×N)
   ├─ RegisterGPUFunctions(db)                 :1175   table functions (gpu_execution, pin_table, …)
   └─ OptimizerExtension::Register(…)          :1645   wire transparent hooks   · enables Step 1
                                                       (hook bodies in transparent/ — see that map)

explicit gpu_execution — Step 1b (per CALL gpu_execution('…')):
GPUExecutionBind()                             :510  ◀─ DuckDB binds the table function (once)
├─ ExtractPlan()                               :241   parse → optimize → resolve
│     └─ PrepareConnection()                   :203   disable incompatible optimizers (explicit-path analogue)
├─ SiriusGeneratePhysicalPlan()                :498   ─▶ plan generator (create_plan)
└─ build sirius_interface + wrap plan          :533/570   → SiriusTableFunctionData (bind state)
GPUExecutionFunction()                         :595  ◀─ DuckDB pulls rows (repeatedly)
├─ if plan_error ─▶ run_internal_cpu_fallback_query   :100   ─▶ DuckDB CPU
├─ iface->sirius_execute_query(…)              :612   ─▶ sirius_interface          · Step 3 → 9
│     └─ on error ─▶ run_internal_cpu_fallback_query  :100   ─▶ DuckDB CPU
└─ data.res->Fetch()                           :632   pull the next chunk
```

The `load` block is a one-time **fan-out** — it *registers* everything, including the
transparent hooks, so `LoadInternal` enables *both* doorways (not a control-flow chain;
shown for orientation). The `explicit gpu_execution` block is the only real chain here, and
its `sirius_execute_query` at `:612` is the **same convergence point** as the transparent
path ([`file-maps/transparent/sirius_optimizer_extension.md`](transparent/sirius_optimizer_extension.md)
→ Call sequence). Any plan/execution error falls back to DuckDB CPU.

## Step 0 — Extension load (runs once at `LOAD`, before any query)

| Function (signature) | Line | Role vs. doc |
|---|---|---|
| `DUCKDB_CPP_EXTENSION_ENTRY(sirius, loader)` | 1674 | C entry point; calls `LoadInternal`. |
| `void SiriusExtension::Load(ExtensionLoader& loader)` | 1657 | Thin wrapper over `LoadInternal`. |
| `static void LoadInternal(ExtensionLoader& loader)` | 1623 | The real bootstrap. **Lines 1645–1648 register `sirius_pre_optimizer_hook` / `sirius_optimizer_hook` — these *are* the doc's Step 1**, but their bodies are in `src/transparent/`, not here. |
| `void SiriusExtension::InitialGPUConfigs(DBConfig& config)` | 1452 | Registers all `SET` options (the `Set*` callbacks at 1246–1450). Not part of query flow. |
| `void SiriusExtension::RegisterGPUFunctions(DatabaseInstance& instance)` | 1175 | Registers the `gpu_execution` table function (and `pin_table`, profiler, etc.). |

## Step 1b — Explicit `CALL gpu_execution(...)` path

The one query-flow region this file genuinely implements. DuckDB calls Bind once,
then Function:

| Function (signature) | Line | Doc step |
|---|---|---|
| `unique_ptr<FunctionData> SiriusExtension::GPUExecutionBind(ClientContext&, TableFunctionBindInput&, vector<LogicalType>&, vector<string>&)` | 510 | **Step 1b.1** — re-parses the inner SQL, optimizes it, generates the Sirius physical plan. Also constructs the `sirius_interface` (line 533). |
| `unique_ptr<LogicalOperator> SiriusTableFunctionData::ExtractPlan(ClientContext&)` | 241 | The parse+optimize+resolve sub-part of 1b.1 (explicit-path equivalent of DuckDB's optimizer running). |
| `static unique_ptr<sirius::op::sirius_physical_operator> SiriusGeneratePhysicalPlan(ClientContext&, unique_ptr<LogicalOperator>&)` | 498 | The "generate the Sirius physical plan" part of 1b.1 — calls `create_plan()`, the **same "single source of truth"** the doc names in Step 1.3 (here at bind-time instead of in `OnFinalizePrepare`). |
| `void SiriusExtension::GPUExecutionFunction(ClientContext&, TableFunctionInput&, DataChunk&)` | 595 | **Step 1b.2** — calls `sirius_iface->sirius_execute_query(...)` at line 611–612. **That call is the handoff to Step 3** (`src/sirius_interface.cpp`); everything past it leaves this file. |
| `unique_ptr<QueryResult> run_internal_cpu_fallback_query(ClientContext&, Connection&, const string&, const string&)` | 100 | **Step 1b.3** — graceful CPU fallback on plan/execution error (also the file's tie-in to the doc's "Error Handling" section). |

**Subtlety while reading:** in the transparent path the doc's Step 1 pre-hook
disables incompatible optimizers (`IN_CLAUSE`, etc.). In this explicit path, that
same disabling happens inside `SiriusTableFunctionData::PrepareConnection` (line
203), called from `ExtractPlan`. Same idea, different trigger.

## Functions *outside* the doc's query flow (don't try to map these)

Separate features or legacy — skip on a first read:

- **S3 rewrite target** (bind-time schema probe; loosely Step 6 scan, not in the
  main trace): `SiriusReadParquetBind` (131), `SiriusReadParquetFunction` (159),
  `SiriusReadParquetCardinality` (168).
- **`pin_table` caching feature** (its own subsystem): `PinTableBind` (791),
  `PinTableFunction` (841), `UnpinTableBind` (1069), `UnpinTableFunction` (1086).
- **Profiler / query-label helpers**: `ProfilerStartFunction` (1114),
  `ProfilerStopFunction` (1126), `SiriusSetQueryLabelBind/Function` (1143/1158).
- **Legacy `gpu_processing` path** (behind `#ifdef SIRIUS_ENABLE_LEGACY` — ignore
  per CLAUDE.md): `GPUProcessingBind` (385), `GPUProcessingFunction` (440),
  `GPUGeneratePhysicalPlan` (371), `GPUBufferInit*` (671/747).

## Takeaway into Week 2

This file is the *doorway*, not the engine. The transparent path (Step 1) is
registered here but implemented in `src/transparent/`; the explicit path (Step 1b)
is implemented here and ends at the `sirius_execute_query()` call on line 612. Both
doorways converge on `sirius_interface.cpp` (Step 3) — your next file. The bulk of
the 1,679 lines (config setters, `pin_table`, registration boilerplate) is exactly
the *skim* material the onboarding tag predicted; the only dense ~130 lines that
matter for the query trace are `GPUExecutionBind` and `GPUExecutionFunction`.

## Types fundamental to *this* file

The generic DuckDB types in these signatures — `ClientContext`, `DataChunk`,
`LogicalType`, `LogicalOperator`, `QueryResult`, `Connection`, `DatabaseInstance`,
`DBConfig`, `TableFunctionBindInput`, `FunctionData`, `TableFunctionInput` — recur
everywhere and are defined once in
[`duckdb-types-glossary.md`](../reference/duckdb-types-glossary.md). Read that first; you'll know
them intimately within a week and won't need them re-explained per file.

What's left below is the short list of types that are *especially* load-bearing for
reading `sirius_extension.cpp` specifically — either because the file is essentially
built around them, or because you mostly only meet them here.

- **The bind → execute contract, as it shapes this whole file.** Almost every
  function here comes in a `…Bind` / `…Function` pair, and that pairing *is* DuckDB's
  table-function contract (see the glossary entries for `TableFunctionBindInput`,
  `FunctionData`, `TableFunctionInput`). Concretely for `gpu_execution`:
  `GPUExecutionBind` (510) runs **once** — it parses/optimizes the inner SQL, calls
  `create_plan()`, and returns a `unique_ptr<FunctionData>`; then
  `GPUExecutionFunction` (595) is called **repeatedly** to stream result chunks,
  receiving that same state back. Once you see this rhythm, the file stops looking
  like 1,679 scattered lines and resolves into ~10 of these pairs. Internalizing the
  contract here is what makes the rest of the file (and later scan/`pin_table` code)
  read quickly.

- **`SiriusTableFunctionData`** *(this file's `FunctionData` subclass; defined at line
  177)* — the concrete "state bundle" threaded from `GPUExecutionBind` to
  `GPUExecutionFunction`. It holds the query text, the CPU-fallback query, the
  generated plan (`sirius_prepared_statement_data`), the `sirius_interface`, and the
  plan-error flags; its `ExtractPlan()` (241) and `PrepareConnection()` (203) methods
  do the parse+optimize work (the latter is where this path disables incompatible
  optimizers — the explicit-path analogue of the transparent pre-hook). **Think:**
  the per-query scratchpad this file fills in during bind so execute has everything it
  needs. Reading it tells you exactly what state a `gpu_execution` call carries.

- **`ExtensionLoader`** — the handle DuckDB hands the extension at `LOAD`, used in
  `LoadInternal` (1623) to register functions and reach the `DatabaseInstance`
  (`loader.GetDatabaseInstance()`). Worth calling out because you meet it **only**
  here — it's the doorway type for Step 0 and essentially never reappears once you
  leave extension registration. **Think:** the installer's toolkit, used for a few
  lines at load time and then gone.

- **`sirius::sirius_interface`** *(usually `sirius_interface`; Sirius's own type)* —
  the GPU engine's front door. Constructed in `GPUExecutionBind` (533) and invoked in
  `GPUExecutionFunction` via `sirius_execute_query(...)` (612). That call is the exact
  boundary where this file's job ends and Step 3 (`sirius_interface.cpp`) begins.
  **Think:** the handoff point — the last Sirius type this file touches before control
  passes to the engine.
