# Sirius Onboarding Study Notes

Working notes from an onboarding study session on the Sirius codebase — a plan plus
supporting explainers, for a contributor who knows modern C++ but not databases.
Generated collaboratively; meant for quick navigation of that discussion.

## Layout

```
sirius-onboarding-study/
├── README.md                     this index
├── onboarding-path.md            the plan (start here)
├── onboarding-coverage-estimate.md
├── weeks/                        one concepts explainer per onboarding week
│   └── week1-concepts.md
├── reference/                    shared references the per-file maps link to
│   └── duckdb-types-glossary.md
└── file-maps/                    one map per source file, MIRRORING sirius/src/
    ├── sirius_extension.md        ↔ src/sirius_extension.cpp
    └── sirius_interface.md        ↔ src/sirius_interface.cpp
```

`file-maps/` mirrors the repo's `src/` tree: a source file at `src/<path>/<name>.cpp`
gets its map at `file-maps/<path>/<name>.md` (e.g. `src/op/sirius_physical_limit.cpp`
→ `file-maps/op/sirius_physical_limit.md`). We add `weeks/` and `file-maps/` entries
as we work through [`onboarding-path.md`](onboarding-path.md); not all exist yet.

## Index

| File | What it is |
|------|------------|
| [onboarding-path.md](onboarding-path.md) | The ~6–7 week onboarding plan: weekly phases, checkable tasks, effort tags, milestones, and a DB-theory resource list. **Start here.** |
| [onboarding-coverage-estimate.md](onboarding-coverage-estimate.md) | How much of the codebase the plan actually has you read — tiered line/percentage estimate (the "spine + deep dives" rationale). |
| [weeks/week1-concepts.md](weeks/week1-concepts.md) | Week 1 checkpoint explainer: physical plan, operator, pipeline, hash join, and how SQL becomes GPU work — cross-referenced to the Week 1 readings. |
| [reference/duckdb-types-glossary.md](reference/duckdb-types-glossary.md) | Shared reference for the recurring DuckDB types (`ClientContext`, `DataChunk`, `LogicalOperator`, …). Per-file maps link here instead of re-explaining. |
| [file-maps/sirius_extension.md](file-maps/sirius_extension.md) | Maps `src/sirius_extension.cpp` onto the steps in `docs/super-sirius/execution-flow.md`, plus the types most fundamental to that file. |
| [file-maps/sirius_interface.md](file-maps/sirius_interface.md) | Maps `src/sirius_interface.cpp` (Steps 3 & 9): the query-lifecycle call graph, the `sirius_active_query` state machine, and the DuckDB methods it forks. |

## Notes

- These notes live **outside** the Sirius repo (one level above it, beside the
  `sirius/` checkout) so they are never tracked or committed.
- These are study aids, not project documentation. Authoritative docs live in
  [`../sirius/docs/super-sirius/`](../sirius/docs/super-sirius/).
- Source/doc paths inside these notes are relative to the **Sirius repo root** at
  `../sirius/` (e.g. `src/op/...` means `../sirius/src/op/...`).
- Line numbers cited in the `file-maps/` notes were accurate as of 2026-06-10;
  re-confirm by grepping the function name if the source has moved.
