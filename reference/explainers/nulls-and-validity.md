# NULLs & Validity

SQL `NULL` is "unknown / missing," and it follows rules that surprise people from a normal
programming background — three-valued logic, propagation, aggregate skipping, join
non-matching. On the GPU it's represented by a separate **validity bitmask**. Both halves
matter the moment you touch expressions (Week 3), and NULL bugs are a classic source of subtle
wrong answers.

> **Prime around:** Week 3 — NULL semantics drive how filters/expressions/aggregates behave;
> a "obvious to DB folks" gap worth closing.

## Three-valued logic (the SQL rules)

A predicate isn't TRUE/FALSE — it's **TRUE / FALSE / UNKNOWN**, where `NULL` makes things
UNKNOWN:

- **Propagation:** almost any operation with a NULL operand yields NULL. `NULL + 1 → NULL`,
  `upper(NULL) → NULL`.
- **Comparisons go UNKNOWN:** `NULL = NULL` is **not** TRUE — it's UNKNOWN. So is `NULL = 5`.
  Use `IS NULL` / `IS NOT NULL` to test for null, and `IS NOT DISTINCT FROM` for a null-safe
  equality (NULL "equals" NULL).
- **`WHERE` keeps only TRUE rows** — UNKNOWN rows are dropped (same as FALSE). This is why
  `WHERE x = NULL` returns nothing.
- **Aggregates skip NULLs:** `SUM`/`AVG`/`MIN`/`MAX` ignore them; `COUNT(x)` counts non-null
  values, while `COUNT(*)` counts rows. (Hence Sirius's `COUNT_VALID` vs `COUNT_ALL` in
  [`aggregation-and-group-by.md`](aggregation-and-group-by.md).)
- **Ordering:** NULLs sort together, position controlled by `NULLS FIRST` / `NULLS LAST`.
- **Joins:** a NULL key does **not** match another NULL (`null_equality::UNEQUAL`) — except a
  null-safe join (`IS NOT DISTINCT FROM`) uses `null_equality::EQUAL`
  ([`hash-join-build-probe.md`](hash-join-build-probe.md)).

## On the GPU: the validity bitmask

cuDF stores nullability **out of band**. A `cudf::column` is:

```
data buffer   : [ 10 | ?? | 30 | 40 ]      values (the slot for a null is undefined)
validity mask : [  1    0    1    1  ]      1 bit per row: 1 = valid, 0 = NULL
null_count    : 1
```

The **validity (null) mask** is an Arrow-style bitmap, one bit per row, separate from the data.
A column with no nulls may omit the mask entirely (cheap fast path). Kernels **propagate** the
mask: an arithmetic kernel ANDs input masks (any null operand → null output); a filter's
predicate produces a boolean column whose *value* is true only where the comparison is TRUE
(UNKNOWN/null → not kept). So three-valued logic on the CPU becomes **mask arithmetic** on the
GPU.

## In Sirius

Columns carry their validity mask through every operator (it rides inside the `cudf::column`
within a `data_batch`), and the operators respect it: filters compact on TRUE only, the
expression executor propagates masks through casts/ops, aggregates use `COUNT_VALID` for
non-null counts, and hash joins pick `null_equality` based on whether the predicate is a
null-safe `IS NOT DISTINCT FROM`. (Sirius even has a regression test for exactly this —
`test/sql/bugfix.test`'s "Issue #56 (IS NOT DISTINCT)".)

## Why it matters

NULL handling is where "looks correct" code gives subtly wrong answers — forgetting that
`NULL = NULL` isn't true, that `WHERE` drops UNKNOWN, or that an aggregate skips nulls. When
reading expression/filter/aggregate code, track the **mask** as carefully as the data; when
*writing* a test ([`testing-sirius.md`](testing-sirius.md)), NULL edge cases are high-value.

## See also

- [`filters-and-expression-evaluation.md`](filters-and-expression-evaluation.md) — predicates
  → boolean column → compaction (where three-valued logic lands);
  [`aggregation-and-group-by.md`](aggregation-and-group-by.md) — `COUNT_VALID`.
- [`hash-join-build-probe.md`](hash-join-build-probe.md) — `null_equality` in joins;
  [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — columns (the mask is part of one).
