# Testing Sirius (SQLLogicTest & unit tests)

Week 3 Days 4–5 is "ship something small" — and the most natural first contribution is a
**test**. Sirius has two test systems; knowing which to use (and how to run just yours)
de-risks the first PR.

> **Prime around:** Week 3 (your first contribution). Commands are in `CLAUDE.md`'s Testing
> section; conventions in `CONTRIBUTING.md`. To run any of this you need a **Linux + NVIDIA**
> box (not a Mac) — see [running Sirius on a cloud GPU](../running-sirius-on-a-cloud-gpu.md).

## Two kinds of tests

### 1. SQLLogicTest — end-to-end SQL correctness

Files: `test/sql/*.test` (e.g. `tpch-sirius.test`, `bugfix.test`). This is DuckDB's
**sqllogictest** format — a `.test` file is a sequence of directives:

```
require sirius                         # load the extension

statement ok                          # a statement expected to succeed (DDL/insert/CALL)
CREATE TABLE t (a INTEGER, b INTEGER);

statement ok
INSERT INTO t VALUES (1, 10), (2, 20);

query I                               # a query returning 1 column ('I' = integer-ish)
SELECT sum(b) FROM t;
----                                  # expected result follows
30
```

`query <coltypes>` declares the result shape (`I` int, `R` real, `T` text — one letter per
column); the rows after `----` are the **expected output**, which the harness diffs against
actual. So a SQLLogicTest asserts *"this SQL gives exactly this answer."* (TPC-H expected
answers live under `test/answers/`.) Tests invoke Sirius via plain SQL or a `CALL` — some older
`.test` files use the legacy `gpu_processing`/`gpu_buffer_init`; the result-matching is the same.

**Run them:**
```
make test                                                    # all SQLLogicTests
build/release/test/unittest --test-dir . test/sql/tpch-sirius.test   # one file
```

### 2. Catch2 — C++ unit tests (component level)

Files: `test/cpp/<component>/` (e.g. `operator/`, `expression_executor/`, `memory/`, `scan/`).
Built into one binary; tests use Catch2 (`TEST_CASE` / `SECTION` / tags).

```
build/release/extension/sirius/test/cpp/sirius_unittest                 # all
build/release/extension/sirius/test/cpp/sirius_unittest "[cpu_cache]"   # by tag
build/release/extension/sirius/test/cpp/sirius_unittest "test_name…"    # by name
```

(Logs: `build/release/extension/sirius/test/cpp/log`.) A unit test exercises an operator,
kernel, or component **in isolation** — more setup, but it can check internals a SQL query
can't.

## Which to write for a first PR

- **SQLLogicTest** is usually the lowest-friction first contribution: pick a query case that
  should work (or reproduce a fixed bug), add the SQL + expected result to a `.test` under
  `test/sql/`, and you've got a regression test. Great for "extend an operator's supported
  cases" or "confirm this query result."
- **Catch2** when you're testing a specific operator/kernel/component you touched in code —
  add a case under `test/cpp/<component>/`.

## The first-PR loop

```
edit/add test → build  (CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make)
             → run just your test (commands above)
             → pre-commit run -a        # clang-format, black, cmake-format, codespell
             → open PR                  # see CONTRIBUTING.md
```

Running *only* your test (the single-file / by-name forms) keeps the loop fast — a full
`make test` is slow.

## See also

- [`onboarding-path.md`](../../onboarding-path.md) — Week 3 Days 4–5 (the PR task).
- [`duckdb-extension-api.md`](duckdb-extension-api.md) — `require sirius` loads the extension
  these tests exercise; [`types-duckdb-cudf-sirius.md`](types-duckdb-cudf-sirius.md) /
  [`nulls-and-validity.md`](nulls-and-validity.md) — the kinds of edge cases worth a `.test`.
