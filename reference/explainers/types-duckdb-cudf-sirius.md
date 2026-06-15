# Types: DuckDB ↔ cuDF ↔ Sirius

A value's *type* drives everything in execution — which kernel runs, how it's stored, when a
cast is needed, and a surprising number of Sirius's GPU footguns. Three type systems meet in
Sirius, and knowing how they map (and where they *don't*) is the foundation for Week 3's
expression work.

> **Prime around:** Week 3 (expressions/casts are type-driven) — but it explains footguns you
> already saw in the aggregate map, and recurs everywhere.

## The three type systems

| System | Type | Examples | Where |
|---|---|---|---|
| **DuckDB** | `LogicalType` | `INTEGER`, `BIGINT`, `HUGEINT`, `DECIMAL(p,s)`, `VARCHAR`, `DATE`, `TIMESTAMP` | the SQL types, from the parser/plan |
| **Sirius** | `sirius::logical_type` (+ `type_id`) | same set, lightweight value type | `src/include/helper/logical_type.hpp`; operators carry `vector<sirius::logical_type>` |
| **cuDF** | `cudf::data_type` (+ `type_id`) | `INT32`, `INT64`, `DECIMAL64`, `STRING`, `TIMESTAMP_*` | the actual GPU **column** types |

Flow: DuckDB hands plans typed as `LogicalType`; Sirius mirrors them as `sirius::logical_type`
(its own SQL type descriptor); and at the GPU boundary they become `cudf::data_type`. The
conversions live in `src/include/helper/type_conversions.hpp` (e.g. `to_duckdb_vec`) and a
`GetCudfType(LogicalType)` mapping (used as `ToCudfType` in the aggregate operator).

## Supported types (and what falls back)

Sirius's GPU path supports: **INTEGER, BIGINT, FLOAT, DOUBLE, VARCHAR, DATE, TIMESTAMP,
DECIMAL** (per `CLAUDE.md`). **Nested types** (`STRUCT`, `LIST`) and some temporal types aren't
supported and trigger **CPU fallback** — the `sirius::type_id` enum lists them (`STRUCT`,
`LIST`), but the plan generator/operators reject them.

## DECIMAL: scale, precision, fixed-point

`DECIMAL(p, s)` is an exact type with **precision** `p` (total digits) and **scale** `s`
(digits after the point). cuDF represents it as **fixed-point**: a scaled integer
(`decimal32`/`decimal64`/`decimal128` backed by `int32`/`int64`/`__int128`). So `100.00`
DECIMAL(10,2) is the integer `10000` with scale `-2`. Two consequences you saw in the
aggregate map:
- cuDF requires **output type == input type** for fixed-point reductions.
- `SUM` **widens** the decimal first (`DECIMAL32→64→128`) to avoid overflow; the final
  precision is applied after.

## The int128 / HUGEINT footgun (the big one)

cuDF has `__int128` **only** as the storage behind `decimal128` — **not** as a general numeric
column type with arithmetic/reduction kernels. So 128-bit integer math isn't available on the
GPU. That single gap causes the recurring fallbacks:
- **`HUGEINT`** (DuckDB's 128-bit int) is **downcast to `BIGINT`** where possible.
- **`BIGINT` (`INT64`) `SUM` falls back to CPU** — summing 64-bit ints needs a 128-bit
  accumulator to avoid silent overflow, which the GPU lacks.
- **`BIGINT` arithmetic (`+`, `-`, `*`) also falls back to CPU**, for the same reason
  ([`filters-and-expression-evaluation.md`](filters-and-expression-evaluation.md)).

When you see a query mysteriously run on CPU, an INT64/HUGEINT path is the usual suspect.

## Casting

Type mismatches are resolved by `cudf::cast` (e.g. small-int → INT64 widening before AVG, or
decimal widening before SUM). The expression executor inserts casts as it evaluates; the hash
join carries `key_casts` to align join-key types on both sides before hashing
([`hash-join-build-probe.md`](hash-join-build-probe.md)). Casts cost a pass over the column, so
they're not free — another reason types matter for performance, not just correctness.

## Why this matters for Week 3

Expression evaluation *is* type manipulation: every `cudf::` call needs the right
`cudf::data_type`, decimals need scale handling, and INT64 paths quietly defect to the CPU.
Holding the DuckDB ↔ Sirius ↔ cuDF mapping (and the int128 gap) in your head is what makes the
expression-executor code — and the "why did this fall back?" questions — make sense.

## See also

- [`filters-and-expression-evaluation.md`](filters-and-expression-evaluation.md) — where
  casts/types drive evaluation; [`aggregation-and-group-by.md`](aggregation-and-group-by.md) —
  decimal widening + the BIGINT-SUM fallback in code.
- [`cudf.md`](cudf.md) — `cudf::column`/`data_type`; [`reference/duckdb-types-glossary.md`](../duckdb-types-glossary.md)
  — `LogicalType`.
