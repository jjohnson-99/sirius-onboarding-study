# Apache Iceberg (open table format)

Sirius can scan **Iceberg** tables (`ICEBERG_SCAN`), so it's worth knowing what Iceberg is:
a **metadata layer that makes a pile of Parquet files in object storage behave like a real,
transactional table.** It's not a file format — it's a *table* format that sits *on top of*
file formats like Parquet ([`parquet-format.md`](parquet-format.md)).

> **Prime around:** Week 5 (the scan subsystem) — and specifically when working on
> `ICEBERG_SCAN` / Iceberg deletes. `docs/super-sirius/scan.md`, `operators.md`.

## The problem it solves

A "data lake" is just files (Parquet) in a bucket (S3/GCS). That's cheap and open, but a
bare pile of files has no **table semantics**: no atomic commits, no consistent reads while
someone's writing, no efficient row-level deletes/updates, no schema/partition changes, no
way to query "the table as of last Tuesday." Iceberg adds a **metadata layer** that fixes
all of that — turning files-in-a-bucket into an ACID, evolvable, time-travelable table.
(Delta Lake and Apache Hudi are the main alternatives.)

## What it provides

- **ACID snapshots.** Each commit produces a new immutable **snapshot** — a consistent set
  of files. Readers see a whole snapshot; writers add a new one atomically.
- **Time travel.** Query an old snapshot ("as of" a timestamp/snapshot id).
- **Schema & partition evolution.** Add/drop/rename columns and even change partitioning
  *without* rewriting data; "hidden partitioning" frees queries from knowing the layout.
- **Row-level deletes/updates.** The hard one on immutable files — done with **delete
  files** rather than rewriting data:
  - **positional deletes** — "row N of file F is deleted,"
  - **equality deletes** — "rows where key = X are deleted,"
  - **deletion vectors** (V3) — a compact bitmap of deleted rows (stored in Puffin files).
  Format versions: **V1** append-only, **V2** positional+equality deletes, **V3** deletion
  vectors.

## The metadata hierarchy

```
catalog
 └ table metadata (metadata.json: schema, partition spec, snapshot list)
    └ snapshot ── manifest list
        └ manifest files (list data + delete files, with per-file stats)
           └ data files (Parquet)  +  delete files
```

A scan resolves the current (or time-traveled) snapshot → manifest list → manifests → the
set of data files and applicable delete files. Per-file stats enable the same row-group
skipping as plain Parquet, plus file-level pruning.

## How Sirius scans Iceberg

`sirius_physical_iceberg_scan` (`ICEBERG_SCAN`) **inherits from the parquet scan** — because
Iceberg's data files *are* Parquet, so the read path is the
[`parquet-format.md`](parquet-format.md) one. What Iceberg adds is **applying deletes**, and
Sirius does it **on the GPU** with no device→host round-trips:

- a delete-filter pipeline composes a **positional-delete filter**, **equality-delete
  filter** (per key-schema, sequence-scoped), and a **deletion-vector filter** (V3);
- **manifest discovery** delegates to DuckDB's `iceberg_metadata()`, with a custom Avro
  reader as the fallback for V3 deletion-vector Puffin files;
- it handles schema evolution, snapshot time-travel, and partition evolution.

Sirius **prefetches** the delete data before pipeline construction (the
`prefetch_iceberg_delete_data` step in
[`file-maps/sirius_engine.md`](../../file-maps/sirius_engine.md)). So the mental model:
**Iceberg = Parquet data files + a metadata/delete layer; Sirius reads the Parquet on the
GPU and applies the Iceberg deletes on the GPU.**

## See also

- [`parquet-format.md`](parquet-format.md) — the file format Iceberg's data files use (and
  that the Iceberg scan inherits).
- [`oltp-vs-olap.md`](oltp-vs-olap.md) — Iceberg is OLAP/data-lake infrastructure;
  [`file-maps/sirius_engine.md`](../../file-maps/sirius_engine.md) — where Sirius prefetches
  Iceberg delete data.
