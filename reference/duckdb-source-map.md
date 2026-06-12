# DuckDB Source Map

Where to look in **DuckDB's own source** for the internals these notes reference. DuckDB
isn't in this study repo (it's the `duckdb/` submodule of the Sirius repo). Paths below are
relative to the **DuckDB repo root** (github.com/duckdb/duckdb).

> **Read at Sirius's pinned commit.** Sirius pins a specific DuckDB submodule version, and
> DuckDB refactors these internals across releases — method names/signatures drift. Treat
> this as a rough map and `grep` within the pinned submodule for the real thing. Complements
> [`duckdb-types-glossary.md`](duckdb-types-glossary.md) and
> [`explainers/duckdb-extension-api.md`](explainers/duckdb-extension-api.md).

## Query lifecycle — the skeleton `sirius_interface` forks

The "based on `ClientContext::…`" comments in `sirius_interface.cpp`
([map](file-maps/sirius_interface.md)) point here:

| DuckDB file | What's there |
|---|---|
| `src/main/client_context.cpp` | `ClientContext::Query`, `PendingStatementOrPreparedStatement`, `PendingPreparedStatementInternal`, `BeginQueryInternal`/`EndQueryInternal` — the exact chain Sirius mirrors |
| `src/main/pending_query_result.cpp` | `PendingQueryResult::ExecuteInternal` — the execute-phase referent |
| `src/include/duckdb/main/client_context.hpp`, `pending_query_result.hpp`, `materialized_query_result.hpp` | the types' declarations |

## Extension API — the doorways

For [`explainers/duckdb-extension-api.md`](explainers/duckdb-extension-api.md) and the
doorway maps:

| DuckDB file | What's there |
|---|---|
| `src/include/duckdb/main/client_context_state.hpp` | `ClientContextState` — base of `SiriusContext`; `QueryBegin`/`QueryEnd`, `OnFinalizePrepare`, `CanRequestRebind` |
| `src/include/duckdb/optimizer/optimizer_extension.hpp` | `OptimizerExtension` — the pre/post hooks behind transparent execution |
| `src/include/duckdb/main/extension_callback.hpp` | `ExtensionCallback` — `OnConnectionOpened`/`Closed` |
| `src/include/duckdb/function/table_function.hpp` | `TableFunction` — the bind/execute contract |
| `src/include/duckdb/main/config.hpp` | `DBConfig` — `AddExtensionOption`, `disabled_optimizers` |

## Logical plans & operators — the plan generator

For [`file-maps/planner/sirius_physical_plan_generator.md`](file-maps/planner/sirius_physical_plan_generator.md):

| DuckDB file | What's there |
|---|---|
| `src/include/duckdb/planner/logical_operator.hpp` | `LogicalOperator` base |
| `src/include/duckdb/planner/operator/list.hpp` | index of every `LogicalXxx` node (the `switch` cases) |
| `src/include/duckdb/common/enums/logical_operator_type.hpp` | `LogicalOperatorType` enum (Sirius's supported/unsupported boundary) |

## Core types — the glossary

For [`duckdb-types-glossary.md`](duckdb-types-glossary.md):

| DuckDB file | What's there |
|---|---|
| `src/include/duckdb/common/types/data_chunk.hpp`, `vector.hpp` | `DataChunk`, `Vector` (the vectorized batch) |
| `src/include/duckdb/common/types.hpp` | `LogicalType` |
| `src/include/duckdb/main/database.hpp` | `DatabaseInstance` |
| `src/include/duckdb/main/prepared_statement_data.hpp` | `PreparedStatementData` |
| `src/include/duckdb/execution/physical_operator.hpp` | `PhysicalOperator` (base of `PhysicalSiriusExecution`) |

## Optimizer — what transparent execution disables

For [`explainers/client-connections.md`](explainers/client-connections.md) (the
disable/restore dance):

| DuckDB file | What's there |
|---|---|
| `src/optimizer/optimizer.cpp` | `Optimizer::Optimize` and the pass list |
| `src/include/duckdb/common/enums/optimizer_type.hpp` | `OptimizerType` — the enum values Sirius disables (`IN_CLAUSE`, `STATISTICS_PROPAGATION`, …) |

## Tip

DuckDB's headers live under `src/include/duckdb/`; implementations under `src/`. If a path
above has drifted, `grep -r "ClassName::Method" duckdb/src` in the Sirius repo (after
`git submodule update --init duckdb`) finds it fast.
