# Shared behaviour

*bberg v0.2.1 · [README](../README.md)*

This page describes behaviour common to [`read_bberg`](bberg.md) and [`read_bbcax`](bbcax.md).

## Loading

```sql
LOAD '/path/to/bberg.linux_amd64.duckdb_extension';
```

The database must allow unsigned extensions (`duckdb -unsigned`, or `allow_unsigned_extensions` in the
client configuration). For Java see [jdbc.md](jdbc.md), for DuckDB-Wasm see [wasm.md](wasm.md).

## Inputs

The first argument selects the input:

| Form | Example | Notes |
|---|---|---|
| File | `'data/prices.out'` | |
| gzip file | `'data/prices.out.gz'` | Decompressed when the name ends in `.gz` and the content starts with the gzip signature. Multi-member archives are supported. |
| Directory | `'data/2026/'` | All files below the directory, recursively |
| Glob | `'data/*/*.out'`, `'data/**/*.cax.gz'` | `*`, `?`, `[...]`, and `**` for any depth. Local paths only. |
| List | `['a.out', 'b.out', 'archive/*.out']` | Entries may be globs or directories. Order is kept and duplicates are removed. |
| Remote | `'s3://bucket/x.out'`, `'https://host/x.cax.gz'` | Requires `httpfs`. Explicit paths or lists only. |

Files matched by one glob or directory are processed in path order. A glob without matches raises
`No files found that match the pattern ...`; a missing file raises DuckDB's `Cannot open file` error.

### Remote storage

All reads go through the file system of the DuckDB instance that loaded the extension. Secrets, credential
chains, proxies and the HTTP cache of that instance apply unchanged.

```sql
INSTALL httpfs;
LOAD httpfs;
CREATE SECRET market_data (TYPE s3, PROVIDER credential_chain, REGION 'eu-west-1');

SELECT count(*) FROM read_bberg('s3://example-bucket/dl/getdata_20260114.out.gz');
```

Remote files are read in 8 MB blocks (local files in 256 KB blocks), which keeps the number of range
requests low. Globs are not expanded on remote paths because DuckDB's C extension API does not expose glob
expansion. Build the file list with DuckDB's `glob()` function instead:

```sql
SET VARIABLE files = (SELECT list(file) FROM glob('s3://example-bucket/dl/*.out.gz'));
SELECT * FROM read_bberg(getvariable('files'), union_by_name = true);
```

### Replacement scan

A quoted path in the `FROM` clause is routed by its extension (case-insensitive, optionally followed by
`.gz`):

| Extension | Function |
|---|---|
| `.out` | `read_bberg` |
| `.cax` | `read_bbcax` |

```sql
SELECT * FROM 'data/prices.out.gz';
SELECT count(*) FROM 'data/*.cax';
```

The replacement scan uses default options. Call the function to pass options.

## Additional columns

- `filename VARCHAR` with `filename = true`: the path as it was opened.
- One `VARCHAR` column per hive partition key with `hive_partitioning = true`. A path such as
  `year=2026/month=01/file.out` adds the columns `year` and `month`. Keys are matched case-insensitively
  across files; a key missing from a path is `NULL` for that file.

## Type detection

Columns named by the data (fields in `.out` files, action parameters in `.cax` files) are typed from a sample
of `sample_size` rows. The first type that all sampled values fit is used. Empty values and NULL markers are
ignored.

| Order | Type | Accepted values |
|---:|---|---|
| 1 | `VARCHAR[][]` | Bulk values, only with `bulk_fields = true` |
| 2 | `DATE` | `2026-01-14`; `20260114` when the header has `DATEFORMAT=yyyymmdd` (years 1900 to 2199) |
| 3 | `TIMESTAMP` | ISO date followed by `[ T]HH:MM[:SS[.ffffff]][Z]`; also columns mixing dates and timestamps |
| 4 | `BOOLEAN` | `true`, `false`, `t`, `f`, `1`, `0` (any case) |
| 5 | `BIGINT` | Integers within 64 bits |
| 6 | `DOUBLE` | Decimal or exponent notation |
| 7 | `VARCHAR` | Anything else |

Rules specific to Bloomberg data:

- Values with leading zeros, such as `00123456` or `007`, keep a column `VARCHAR`. They are identifiers
  (CUSIP, SEDOL and similar) whose zeros would be lost as numbers.
- Placeholders such as `N.A.` keep a column `VARCHAR` unless they are declared as NULL markers with
  `bloomberg_nulls = true` or `nullstr`.
- `detect_dates = false` disables steps 2 and 3. Compact dates are then `BIGINT`, ISO dates `VARCHAR`.
- `all_varchar = true` or `auto_detect = false` disables detection; all such columns are `VARCHAR`.

## Explicit types

- `types = {'PX_LAST': 'DECIMAL(18,6)', 'CP_RECORD_DT': 'DATE'}` sets the type of individual columns by name
  (case-insensitive). All other columns keep detection. An unknown name is an error.
- `columns = {...}` (`read_bberg` only) replaces the complete schema.

| Type names | Notes |
|---|---|
| `VARCHAR`, `TEXT`, `STRING`, `CHAR`, `BPCHAR` | |
| `BOOLEAN`, `BOOL`, `LOGICAL` | `true/t/yes/y/1`, `false/f/no/n/0` |
| `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT`, `INT1`, `INT2`, `INT4`, `INT8`, `INT`, `SHORT`, `LONG`, `SIGNED` | Decimal input is rounded (`1.5` becomes `2`). `_` digit separators are accepted. Out-of-range values are `NULL`. |
| `FLOAT`, `REAL`, `DOUBLE`, `FLOAT4`, `FLOAT8` | |
| `DECIMAL(p,s)`, `NUMERIC(p,s)`, `DECIMAL` | Exact parsing up to `DECIMAL(38,s)`, rounded half away from zero. `DECIMAL` alone means `DECIMAL(18,3)`. Overflow is `NULL`. |
| `DATE` | `YYYY-MM-DD` (separator `-`, `/`, `.` or space); `YYYYMMDD` when `DATEFORMAT=yyyymmdd` |
| `TIMESTAMP`, `DATETIME` | A date as above, optionally followed by `[ T]HH:MM[:SS[.ffffff]]` and `Z` |
| `VARCHAR[][]` | Bloomberg bulk value; values that do not decode are `NULL` |

A value that cannot be converted to its column type becomes `NULL`, consistent with `TRY_CAST`. Number
parsing does not depend on the process locale.

## NULL handling

| Option | Effect |
|---|---|
| `nullstr = 'N.A.'` or `nullstr = ['N.A.', '-']` | Values equal to a marker after trimming become `NULL`. Markers are ignored by type detection, so a numeric column with gaps stays numeric. |
| `bloomberg_nulls = true` | Adds `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A` and `FLD UNKNOWN` to the markers. |
| `empty_as_null = true` | Empty `VARCHAR` values become `NULL`. Empty values in other column types are always `NULL`. |

## Text encoding

Input must be valid UTF-8 by default; ASCII is the common case. An invalid line raises an error naming the
file and line, or is skipped with `ignore_errors = true`. Older feeds in ISO-8859-1 can be read with
`encoding = 'latin-1'` (aliases `latin1`, `iso-8859-1`).

## Parallelism

- Files are scanned in parallel, one file per thread, up to one thread per core.
- Uncompressed files larger than `split_size` (16 MB by default) are also split into byte ranges at line
  boundaries, one range per thread. gzip files are read by a single thread each.
- Row order is therefore not defined unless the query has an `ORDER BY`. `parallel = false` reads on one
  thread, files in input order and rows in file order.
- `CREATE TABLE ... AS` and `INSERT` preserve insertion order by default, which forces a single thread for
  table functions using the C API. When order does not matter, `SET preserve_insertion_order = false;` lets
  the load use all cores. Aggregations and joins run in parallel either way.
- Schema discovery reads every file's header and samples rows until `sample_size` is reached, so it takes
  longer with many small files.
- Across files, `read_bberg` requires identical field lists unless `union_by_name = true`. `read_bbcax` with
  `action` set builds the union of parameter keys automatically.

## Performance

- Select only the columns you need. Unselected columns are not parsed; this matters most for the
  `PARAMETERS` and `CP_PARAMETERS` maps.
- Disable `preserve_insertion_order` before loading tables.
- Fix individual types with `types` instead of increasing `sample_size`.
- Uncompressed local files are fastest because they are split across threads. For remote storage, gzip
  usually wins because less data is transferred.

## Errors

| Situation | Behaviour |
|---|---|
| Data line with too few fields | Error; skipped with `ignore_errors = true` |
| Invalid UTF-8 | Error naming file and line; convert with `encoding = 'latin-1'` or skip with `ignore_errors = true` |
| Different `.out` field lists across files | Error unless `union_by_name = true` |
| Truncated or corrupt gzip | Error (`unexpected end of file (truncated archive)` or the zlib message) |
| Glob without matches | Error |
| Invalid type in `columns` or `types`, unknown `types` column, invalid `encoding`, `split_size <= 0` | Error when the query is bound |
| `store_rejects = true` or `rejects_scan` | Error; reject tables are not supported |
