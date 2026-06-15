# The Parquet Storage Format

Parquet is the on-disk file format Sirius reads **straight off storage into GPU
memory**, so it's worth knowing how it's actually laid out — the structure explains why
the parquet scan works the way it does, why bind can learn a schema without reading
data, and why there's a pile of GPU decoders in `src/cuda/scan/`. Parquet is the on-disk
embodiment of the columnar idea in
[`columnar-vs-row-storage.md`](columnar-vs-row-storage.md).

> **Prime around:** Week 5 (the scan subsystem) — or any time a query you're tracing
> hits the parquet scan.

## The layout, top to bottom

A Parquet file is a **nested hierarchy**, columnar at its core:

```
File
├── (magic "PAR1")
├── Row Group 0            ← a horizontal slice of rows (e.g. ~1M rows)
│   ├── Column Chunk: id     ← all of `id` for those rows, together
│   │   ├── Dictionary Page
│   │   └── Data Page, Data Page, ...   ← unit of encoding + compression
│   ├── Column Chunk: name
│   └── Column Chunk: qty
├── Row Group 1
│   └── ...
└── Footer (FileMetaData)  ← schema, row-group/column offsets, per-chunk stats
    └── (magic "PAR1")
```

Four levels worth holding in your head:

- **Row group** — a horizontal partition of the table's rows, self-contained. This is
  the unit of **parallelism**: independent row groups can be read by different
  threads/tasks. (Sirius partitions parquet scan work by row group — see
  `GPU_PARQUET_SCAN` in `docs/super-sirius/operators.md`, "row groups are partitioned by
  `approximate_batch_size`.")
- **Column chunk** — within one row group, all values of **one column**, stored
  contiguously. *This is the columnar bit*: a query needing only `qty` reads only the
  `qty` column chunks and skips the rest.
- **Page** — a column chunk is a sequence of pages; the page is the **unit of encoding
  and compression**. An optional *dictionary page* holds distinct values; *data pages*
  hold (encoded) values plus the definition/repetition levels that encode nulls and
  nesting.
- **Footer** — Thrift-encoded `FileMetaData` at the end: the schema, the byte offset of
  every row group and column chunk, and per-chunk **statistics** (min, max, null count).

## The footer is the superpower

Because all the metadata lives in the footer, a reader does a tiny seek-to-end read
first and learns *everything about the file's structure* before touching the bulk data.
That single fact enables three things:

1. **Cheap schema discovery** — you get column names/types from the footer alone. This
   is exactly why a table function's **bind** can describe a parquet table without
   reading rows: Sirius's `describe_parquet` reads only the footer (see
   [`duckdb-table-functions.md`](duckdb-table-functions.md) and `SiriusReadParquetBind`).
2. **Column pruning** — the footer gives each column chunk's byte offset, so the reader
   fetches *only* the byte ranges for the columns the query needs. (`PARQUET_SCAN`
   "reads column-chunk byte ranges" — operators.md.)
3. **Row-group skipping (predicate pushdown)** — the per-chunk min/max stats let a reader
   skip an entire row group without decoding it: `WHERE l_shipdate > '1998-01-01'` can
   drop every row group whose `l_shipdate` max is below that date. Less data read, less
   decoded.

## Encoding & compression (and why Sirius has GPU decoders)

Within a page, values are first **encoded** (logical → compact bytes), then optionally
**compressed** (e.g. snappy, zstd). Common encodings, all leaning on the
single-type-per-column property from
[`columnar-vs-row-storage.md`](columnar-vs-row-storage.md):

- **Dictionary** (`RLE_DICTIONARY`) — store distinct values once, then store small
  indices into that dictionary. Great for low-cardinality columns (e.g. `l_returnflag`).
- **RLE / bit-packing** — run-length-encode repeats; pack small integers into the minimum
  bits.
- **Delta** (`DELTA_BINARY_PACKED`, `DELTA_BYTE_ARRAY`) — store differences between
  consecutive values; great for sorted keys / timestamps.
- **ALP**, byte-stream-split — specialized floating-point encodings.

Decoding all of this is itself parallelizable over a column, so Sirius does it **on the
GPU** rather than on the CPU — that's what the `src/cuda/scan/` kernels are:
`gpu_decode_bitpacking.cu`, `gpu_decode_rle.cu`, `gpu_decode_alp.cu`,
`gpu_decode_strings.cu`, `gpu_native_decode.cu`. The scan reads compressed column-chunk
bytes off disk and decompresses/decodes them directly into `cudf::column`s on the
device.

## Why analytics engines love Parquet

Pulling it together, Parquet is purpose-built for OLAP scans:

- **Columnar** → read only needed columns; excellent compression.
- **Row groups** → independent, splittable units of parallel work.
- **Footer stats** → skip row groups via min/max; cheap schema without scanning.
- **Self-describing & portable** → schema travels with the data; readable by DuckDB,
  Spark, cuDF, pandas, etc.

## How Sirius uses it (the through-line)

`SELECT l_quantity FROM read_parquet('lineitem.parquet') WHERE l_quantity > 10`:

1. **Bind** reads the footer → schema (`describe_parquet`); the optimizer plans against
   it.
2. The pipeline splits the scan into per-**row-group** tasks (`GPU_PARQUET_SCAN`).
3. Each task fetches only the `l_quantity` **column-chunk** byte ranges (column pruning),
   possibly skipping row groups whose stats can't satisfy `> 10`.
4. GPU **decoders** (`src/cuda/scan/`) decompress those bytes into a `cudf::column` on
   the device — no CPU round-trip.
5. From there it's an ordinary GPU pipeline (filter → … ), exactly the operator flow from
   [`weeks/week2-concepts.md`](../../weeks/week2-concepts.md).

Full scan-subsystem detail is Week 5 (`docs/super-sirius/scan.md`); this is the on-disk
shape that subsystem is built around.

## See also

- [`columnar-vs-row-storage.md`](columnar-vs-row-storage.md) — *why* columnar; Parquet is
  its on-disk form.
- [`duckdb-table-functions.md`](duckdb-table-functions.md) — how footer-only bind fits the
  bind/execute contract.
