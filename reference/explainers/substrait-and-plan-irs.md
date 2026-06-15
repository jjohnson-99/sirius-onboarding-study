# Substrait & Plan IRs

A query plan needs a *representation* as it travels from the front end (parse + optimize) to
the execution engine. That representation is a **plan IR** (intermediate representation), and
**Substrait** is the open, portable standard for one. Sirius mostly uses DuckDB's *in-memory*
plan directly rather than Substrait — but the concept explains the plan generator, and the
repo still vendors a `substrait` submodule, so it's worth knowing the distinction.

> **Prime around:** Week 2 — context for [`file-maps/planner/sirius_physical_plan_generator.md`](../../file-maps/planner/sirius_physical_plan_generator.md)
> (why Sirius translates DuckDB plans the way it does); also ecosystem orientation with
> [`rapids-gpu-data-ecosystem.md`](rapids-gpu-data-ecosystem.md).

## What a plan IR is

Recall the pipeline (from [`weeks/week1-concepts.md`](../../weeks/week1-concepts.md)): SQL → parse →
**logical plan** → optimize → **physical plan** → execute. A **plan IR** is the engine-neutral
representation of the plan that sits *between* the front end and the back end. Two flavors:

- **In-memory IR** — a tree of plan-node objects inside one engine (e.g. DuckDB's
  `LogicalOperator` tree). Fast, but engine-specific and not portable.
- **Portable / serialized IR** — a language- and engine-neutral *format* you can serialize,
  ship over the wire, and have a *different* engine consume. **Substrait** is this.

The point of a portable IR: **decouple the front end from the execution engine** so they can
mix and match — exactly the composable-data-systems idea in
[`rapids-gpu-data-ecosystem.md`](rapids-gpu-data-ecosystem.md).

## What Substrait is

**Substrait** is an open, cross-language **specification for relational query plans** (a
protobuf-based format). It's a "lingua franca for query plans": one system *produces* a
Substrait plan and another *consumes* it.

- **Producers** (front ends that emit Substrait): DuckDB (via extension), Spark, Apache
  Calcite, Ibis, …
- **Consumers** (engines that execute Substrait): DataFusion, Velox/Acero, …

So you could optimize a query in DuckDB and execute it in Velox or a GPU engine, without
either side knowing about the other — the Substrait plan is the contract.

## Sirius's relationship to Substrait (honest version)

The repo vendors a `substrait` submodule (a `sirius-db` fork of the DuckDB Substrait
extension), and Substrait shows up in two ways — but **only one path ever used it**:

- **Legacy `gpu_processing`:** DuckDB logical plan → **Substrait** → GPU plan. Substrait was
  the bridge IR (the legacy `GPUProcessingSubstrait*` entry points).
- **Super Sirius `gpu_execution` (current):** the plan generator walks DuckDB's
  `LogicalOperator` tree **directly** into a `sirius_physical_operator` tree — **no
  Substrait**. The leftover `// #include "from_substrait.hpp"` and the "would complicate the
  conversion to Substrait" optimizer-disable comments in `sirius_extension.cpp` are
  **vestigial** (copied from / shared with the legacy path).

So the live "IR" Super Sirius actually consumes is **DuckDB's own in-memory `LogicalOperator`
tree** — a plan IR, just DuckDB-specific and not serialized. Substrait is historical context
plus an *optional* portability layer Sirius isn't currently on the critical path for.

Why DuckDB skipped the portable IR here: Sirius is a DuckDB *extension* — it's already inside
DuckDB's process with direct access to the optimized `LogicalOperator` tree, so translating it
in memory is simpler and lossless than round-tripping through Substrait. A portable IR earns
its keep when front end and back end are *separate* systems; here they share an address space.

## Takeaway

When you read the plan generator, you're reading a **direct DuckDB-IR → Sirius-IR
translation** (in-memory `LogicalOperator` → `sirius_physical_operator`); Substrait is the
*portable* alternative the legacy path used and the ecosystem standard for plan interchange.
Knowing that distinction explains the vestigial Substrait bits and why the `substrait`
submodule exists without being central.

## See also

- [`file-maps/planner/sirius_physical_plan_generator.md`](../../file-maps/planner/sirius_physical_plan_generator.md)
  — the direct `LogicalOperator` → `sirius_physical_operator` translation.
- [`rapids-gpu-data-ecosystem.md`](rapids-gpu-data-ecosystem.md) — composable data systems,
  where portable plan IRs (Substrait) fit; [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md)
  — `LogicalOperator` (DuckDB's in-memory IR).
