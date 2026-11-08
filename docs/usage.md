# Using the bberg readers

*Documentation for `bberg` v0.2.1. [Back to the README](../README.md).*

This page covers the behaviour `read_bberg` and `read_bbcax` share. Format-specific details live in
[bberg.md](bberg.md) and [bbcax.md](bbcax.md).

## Loading

```sql
LOAD '/path/to/bberg.linux_amd64.duckdb_extension';   -- needs allow_unsigned_extensions / duckdb -unsigned
```

The extension works with DuckDB 1.5.6 and newer, including 2.0.x. See
[DuckDB version compatibility](../README.md#duckdb-version-compatibility). For DuckDB-Wasm (browser/Node) see
[wasm.md](wasm.md): there, inputs must be explicit paths or lists, because globs and directories are not
available.

## Inputs

The first argument is the input. It can be any of the following:

| Form | Example | Notes |
|---|---|---|
| single file | `'data/prices.out'` | |
| gzip file | `'data/prices.out.gz'` | decoded when the name ends in `.gz` **and** the file starts with the gzip magic bytes; a `.gz`-named plain file is read as text; multi-member gzip is supported |
| directory | `'data/2026/'` | every file below it, recursively |
| glob | `'data/*/*.out'`, `'data/**/*.cax.gz'` | `*`, `?`, `[...]`, and `**` for any depth. Local paths only |
| list | `['a.out', 'b.out', 'archive/*.out']` | each entry may itself be a glob or directory; the list order is kept, and duplicate files are dropped |
| remote | `'s3://bucket/x.out'`, `'https://host/x.cax.gz'` | needs `httpfs`; explicit paths (or lists of them), not globs |

Within one glob or directory, files are sorted by path. A glob that matches nothing is an error
(`No files found that match the pattern ...`). A missing explicit path fails with DuckDB's own
`Cannot open file` error.

### Remote storage (S3, GCS, Azure via httpfs, HTTP)

Every byte is read through the file system of the DuckDB instance that loaded the extension. Whatever that
DuckDB can open, these readers can open too, using the same secrets, credential chains, proxies and caching:

```sql
INSTALL httpfs; LOAD httpfs;
CREATE SECRET bbg (TYPE s3, PROVIDER credential_chain, REGION 'eu-west-1');

SELECT count(*) FROM read_bberg('s3://my-bucket/dl/getdata_20260114.out.gz');
SELECT * FROM read_bbcax(['s3://my-bucket/cax/a.cax.gz', 's3://my-bucket/cax/b.cax.gz'], action='DVD_CASH');
```

Remote files are read in 8 MB blocks (local files in 256 KB blocks), so a large S3 object costs few range
requests. Globs are not expanded on remote paths, because DuckDB's C extension API has no glob function.
To read many remote files, build the list in SQL first, for example from DuckDB's own `glob()`:

```sql
SET VARIABLE files = (SELECT list(file) FROM glob('s3://my-bucket/dl/*.out.gz'));
SELECT * FROM read_bberg(getvariable('files'), union_by_name=true);
```

### Replacement scan

A quoted path in `FROM` picks the reader from the file extension (case-insensitive, optionally followed by
`.gz`):

| Extension | Reader |
|---|---|
| `.out` | `read_bberg` |
| `.cax` | `read_bbcax` |

```sql
SELECT * FROM 'data/prices.out.gz';
SELECT count(*) FROM 'data/*.cax';
```

The replacement scan always uses default options. Call the function directly to pass options.

## Output columns

Every reader produces its format's columns (see the per-function pages), followed by:

- `filename VARCHAR`, when `filename=true`: the path exactly as it was opened.
- One `VARCHAR` column per hive partition key, when `hive_partitioning=true`. Path segments such as
  `year=2026/month=11` become columns `year` and `month`. Keys are matched case-insensitively across files,
  and a key missing from a file's path is `NULL` for that file's rows.

`read_bberg` and `read_bbcax` also start with `PARAMETERS MAP(VARCHAR, VARCHAR)`: the `KEY=VALUE` header of the
**file the row came from** (`RUNDATE`, `PROGRAMNAME`, `DATEFORMAT`, `TIMESTARTED`, ...).

## Types

### Automatic detection

Columns whose names come from the data (Bloomberg fields, `.cax` action parameters) are typed from a sample
(`sample_size`). The first type that every sampled, non-empty, non-NULL-marker value fits is chosen:

```
DATE       ISO 2026-01-14, or 20260114 in files whose header says DATEFORMAT=yyyymmdd (years 1900-2199)
TIMESTAMP  ISO date + [ T]HH:MM[:SS[.ffffff]][Z]  (a column mixing dates and timestamps)
BOOLEAN    true/false/t/f/1/0, any case
BIGINT     integers that fit in 64 bits
DOUBLE     decimal or exponent notation
VARCHAR    anything else
```

- Values with leading zeros, such as `00123456` or `007`, keep a column `VARCHAR`. In Bloomberg exports they
  are identifiers (CUSIP, SEDOL, ...) whose zeros would be lost as numbers.
- Bloomberg's `N.A.` / `N.S.` placeholders make a column `VARCHAR` unless they are declared NULL markers
  (`bloomberg_nulls=true` or `nullstr`); declared markers are ignored by detection.
- `detect_dates=false` skips `DATE`/`TIMESTAMP` (compact dates are then `BIGINT`, ISO dates `VARCHAR`).
- `bulk_fields=true` adds a first candidate, `VARCHAR[][]`, for Bloomberg bulk values (see
  [bberg.md](bberg.md#bulk-fields)).
- `all_varchar=true` (or `auto_detect=false`) turns detection off, so every detected column is `VARCHAR`.

### Explicit types

- `types={'PX_LAST': 'DECIMAL(18,6)', 'CP_RECORD_DT': 'DATE'}` (both readers) sets the type of individual
  columns by name (case-insensitive) and leaves the rest to detection. An unknown name is an error.
- `columns={...}` (`read_bberg` only) replaces the whole schema: every column, in order, with its type.

| Type names | Notes |
|---|---|
| `VARCHAR`, `TEXT`, `STRING`, `CHAR`, `BPCHAR` | |
| `BOOLEAN`, `BOOL`, `LOGICAL` | `true/t/yes/y/1`, `false/f/no/n/0` |
| `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT` (+ `INT1/2/4/8`, `INT`, `SHORT`, `LONG`, `SIGNED`) | decimals are rounded (`'1.5'` becomes 2); `_` digit separators are allowed; out-of-range values are `NULL` |
| `FLOAT`, `REAL`, `DOUBLE`, `FLOAT4`, `FLOAT8` | |
| `DECIMAL(p,s)`, `NUMERIC(p,s)`, `DECIMAL` | exact parsing up to `DECIMAL(38,s)`, rounded half away from zero at scale `s`. `DECIMAL` alone is `DECIMAL(18,3)`. Values that overflow are `NULL` |
| `DATE` | `YYYY-MM-DD` (also with `/`, `.` or space as separator), and `YYYYMMDD` when the file header has `DATEFORMAT=yyyymmdd` |
| `TIMESTAMP`, `DATETIME` | a date as above, optionally followed by `[ T]HH:MM[:SS[.ffffff]]` and `Z` |
| `VARCHAR[][]` | a Bloomberg bulk value decoded into rows of cells; values that do not decode are `NULL` |

A value that does not parse as its column's type becomes `NULL`, the way DuckDB's `TRY_CAST` behaves. All
number parsing is independent of the process locale.

## NULL handling

| Option | Effect |
|---|---|
| `nullstr='N.A.'` or `nullstr=['N.A.', '-']` | trimmed values equal to a marker become `NULL`; markers are also ignored by type detection, so a numeric column with `N.A.` gaps is still numeric |
| `bloomberg_nulls=true` | adds `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A`, `FLD UNKNOWN` to the markers |
| `empty_as_null=true` | empty `VARCHAR` values become `NULL` (an empty value in a non-`VARCHAR` column is always `NULL`) |

## Text encoding

Text must be valid UTF-8 (ASCII is the common case). A line with invalid UTF-8 is an error naming the file
and line, or is skipped with `ignore_errors=true`. Files in ISO-8859-1 (Latin-1), which older Bloomberg feeds
use for accented issuer names, are converted with `encoding='latin-1'` (aliases `latin1`, `iso-8859-1`).

## Multiple files and parallelism

- Files are scanned in parallel, one file per thread, with up to one thread per core.
- A large uncompressed file is also split: its data section is cut into `split_size` byte ranges (default
  16 MB) at line boundaries, and each range goes to its own thread. gzip files are read by one thread each,
  because a compressed stream cannot be entered in the middle.
- Row order is therefore unspecified unless you `ORDER BY`. `parallel=false` reads everything on one thread,
  files in input order and rows in file order.
- **`CREATE TABLE ... AS` / `INSERT` run on one thread by default.** DuckDB keeps insertion order
  (`preserve_insertion_order=true`), and a C-API table function cannot provide the batch indexes it would
  need to do that in parallel. When the order does not matter, `SET preserve_insertion_order = false;`
  makes these loads use all cores (about 4x faster on a 12-core machine). Aggregations and joins are
  parallel either way.
- Bind (schema discovery) reads the header of every file and samples rows until `sample_size` is reached. For
  thousands of small files, bind cost grows with the file count.
- Schemas across files:
  - `read_bberg` requires identical `START-OF-FIELDS` lists unless `union_by_name=true`. With union, fields are
    matched by name (case-insensitive, any order).
  - `read_bbcax` (with `action=`) builds the union of discovered keys automatically.

## Performance tips

- **Select only the columns you need.** Projection pushdown means unselected columns are never parsed. In
  particular, the per-row `PARAMETERS`/`CP_PARAMETERS` maps are expensive and cost nothing when not selected.
- **Loading a table?** `SET preserve_insertion_order = false;` first, so the load runs in parallel.
- **Pin types with `types=`** when detection guesses wrong, rather than raising `sample_size` (a larger
  sample makes bind read more data).
- **gzip costs CPU and parallelism.** Uncompressed local files are fastest and are split across threads. For
  remote storage, gzip usually wins because it moves fewer bytes.

## Errors

| Situation | Behaviour |
|---|---|
| malformed `.out`/`.cax` data row (too few fields) | error, or skipped with `ignore_errors=true` |
| invalid UTF-8 | error naming the file and line; `encoding='latin-1'` converts, `ignore_errors=true` skips the line |
| mismatched `.out` field lists across files | error unless `union_by_name=true` |
| truncated or corrupt gzip | error (`unexpected end of file (truncated archive)` / zlib message) |
| glob with no matches | error |
| invalid `columns=`/`types=` type, unknown `types=` column, `nullstr`/`action` type, `encoding`, `split_size <= 0` | bind-time error |
| `store_rejects=true` / `rejects_scan` | error: reject tables are not supported |
