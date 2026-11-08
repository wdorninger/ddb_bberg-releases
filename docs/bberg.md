# `read_bberg`

*Documentation for `bberg` v0.2.0. [Back to the README](../README.md).*

> Paths such as `test/sql/data/...` in the examples refer to the extension's own test files, which are not published. Use your own files.

## Overview

`read_bberg` reads Bloomberg Data License pipe-delimited `.out` exports (plain text or gzip-compressed), in both
the per-security layout and the historical (`hist=true`) layout. Behaviour shared with the other readers
(inputs, remote files, types, NULLs, parallelism) is described in [usage.md](usage.md).

`path` is a file, directory, glob, or a `LIST` of those (`read_bberg(['a.out', 'b/*.out'])`). A bare
`.out` / `.out.gz` path in `FROM` is read with `read_bberg` automatically (replacement scan):

```sql
SELECT * FROM 'test/sql/data/year=2026/month=11/sample_bberg.out';
```

## Quick start

```sql
LOAD '/path/to/bberg.linux_amd64.duckdb_extension';

SELECT *
FROM read_bberg('test/sql/data/year=2026/month=11/sample_bberg.out')
LIMIT 5;
```

## `.out` file capabilities

- Reads Bloomberg export sections (`START-OF-FILE`, `START-OF-FIELDS`, `START-OF-DATA`, etc.).
- Supports plain `.out` and `.out.gz` (gzip is detected by the `.gz` suffix plus the gzip magic bytes; a
  truncated archive is an error).
- Reads through DuckDB's own file system, so remote paths (`s3://`, `https://`, `gcs://`, ... with `httpfs`
  loaded and its secrets/credentials) work like local ones. Remote files are read in 8 MB blocks.
- Accepts file globs (for example `*.out`, `*.out.gz`, `**/*.out`) and directory inputs. Glob expansion and
  directory listing work on local paths only (the C extension API has no glob); remote paths must be explicit
  file paths (or a `LIST` of them). A glob that matches nothing is an error.
- Recursively scans directories and returns a unified stream.
- Extracts each file's header key/value pairs into a `PARAMETERS` map column (per file: a row always carries
  the header of the file it came from).
- Optionally extracts hive-style partition columns from paths like `year=2026/month=11/...`.
- Streams data in chunks at execution time (does not fully materialize files before query processing).
- Scans run in parallel: one thread per file, and large uncompressed files are additionally split into
  `split_size` byte ranges, one per thread. Row order is then unspecified unless you `ORDER BY` (use
  `parallel=false` for file order).
- Only the columns a query uses are parsed (projection pushdown), e.g. `PARAMETERS` costs nothing unless
  selected.

## Output schema

`read_bberg` always includes:

- `PARAMETERS MAP(VARCHAR, VARCHAR)`
- `SECURITY_DESC VARCHAR`
- `ERROR_CODE BIGINT`
- `NUM_FIELDS BIGINT`
- one column per declared field in the file's `START-OF-FIELDS` section

When `hist=true`, `read_bberg` also adds:

- `DTE DATE` (between `NUM_FIELDS` and dynamic field columns)

Optional extra output columns:

- `filename VARCHAR` when `filename=true`
- one `VARCHAR` column per detected hive partition key when `hive_partitioning=true`

## `read_bberg` options (all flags)

All options below are accepted by `read_bberg(path, ...)`.

| Option | Type | Default | Applies to | Behavior |
|---|---|---|---|---|
| `auto_detect` | `BOOLEAN` | `true` | `.out`, historical `.out` | Infer dynamic field types from sampled rows; fixed columns keep their fixed types. |
| `comment_prefix` | `VARCHAR` | `'#'` | `.out`, historical `.out` | Skip lines starting with this prefix in `FIELDS` and `DATA` sections. |
| `filename` | `BOOLEAN` | `false` | `.out`, historical `.out` | Appends a `filename` output column. |
| `empty_as_null` | `BOOLEAN` | `false` | `.out`, historical `.out` | Converts empty strings to `NULL`. |
| `union_by_name` | `BOOLEAN` | `false` | `.out`, historical `.out` | Merge schemas from multiple files into a superset; columns absent in a given file are filled with `NULL`. When `false` (default), all files must declare identical fields or an error is raised. |
| `columns` | `STRUCT` | unset | `.out`, historical `.out` | Override output names/types. Must list exactly all data columns (`SECURITY_DESC`, `ERROR_CODE`, `NUM_FIELDS`, [`DTE`,] fields) in order, otherwise an error is raised. Supported types: `VARCHAR`, `BOOLEAN`, `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT`, `FLOAT`, `DOUBLE`, `DECIMAL(p,s)` (exact; `DECIMAL` alone is `DECIMAL(18,3)`), `DATE`, `TIMESTAMP`, `VARCHAR[][]` (bulk field) and their usual aliases. `DATE`/`TIMESTAMP` accept ISO dates and, when the header says `DATEFORMAT=yyyymmdd`, compact `20260114`. |
| `types` | `STRUCT` | unset | `.out`, historical `.out` | Override the type of individual columns by name, e.g. `types={'PX_LAST': 'DECIMAL(18,6)'}`; all other columns keep detection. Names match the field or output name case-insensitively; an unknown name is an error. Same type names as `columns`. Cannot be combined with `columns`. Wins over `all_varchar`. |
| `detect_dates` | `BOOLEAN` | `true` | `.out`, historical `.out` | Detect `DATE` / `TIMESTAMP` columns (see Type detection). `false` keeps such columns `BIGINT` / `VARCHAR`. |
| `bulk_fields` | `BOOLEAN` | `false` | `.out`, historical `.out` | Decode bulk fields (`;2;rows;cols;...`) into `VARCHAR[][]` (see Bulk fields). |
| `encoding` | `VARCHAR` | `'utf-8'` | `.out`, historical `.out` | `'utf-8'` validates text (invalid UTF-8 is an error, or skipped with `ignore_errors`); `'latin-1'` (`latin1`, `iso-8859-1`) converts ISO-8859-1 files. |
| `split_size` | `BIGINT` | `16777216` | `.out`, historical `.out` | Byte size of the ranges an uncompressed file larger than this is split into for parallel scanning. |
| `all_varchar` | `BOOLEAN` | `false` | `.out`, historical `.out` | Forces inferred data columns to `VARCHAR` (except fixed typed columns). |
| `sample_size` | `BIGINT` | `20480` | `.out`, historical `.out` | Number of rows sampled for type inference. Values `<= 0` mean "sample all available rows" during bind. |
| `hive_partitioning` | `BOOLEAN` | `false` | `.out`, historical `.out` | Extract path segments `key=value` into output columns (as `VARCHAR`). |
| `ignore_errors` | `BOOLEAN` | `false` | `.out`, historical `.out` | Skips malformed data rows with fewer than 3 columns and lines that are not valid UTF-8; otherwise query errors out. |
| `parallel` | `BOOLEAN` | `true` | `.out`, historical `.out` | Scan files, and byte ranges of large uncompressed files, on multiple threads. `false` reads everything on one thread in file order. |
| `store_rejects` | `BOOLEAN` | `false` | `.out`, historical `.out` | Not supported: `true` is an error (use `ignore_errors`). Accepted for `read_csv` compatibility. |
| `rejects_scan` | `VARCHAR` | unset | `.out`, historical `.out` | Not supported: setting it is an error. Accepted for `read_csv` compatibility. |
| `nullstr` | `VARCHAR` or `LIST<VARCHAR>` | unset | `.out`, historical `.out` | Any trimmed value equal to one of these markers is returned as `NULL`, and is ignored by type detection. |
| `bloomberg_nulls` | `BOOLEAN` | `false` | `.out`, historical `.out` | Adds Bloomberg's "no value" markers to `nullstr`: `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A`, `FLD UNKNOWN`. |
| `hist` | `BOOLEAN` | `false` | historical `.out` | Enables historical layout handling by inserting `DTE DATE` and applying historical date parsing rules. |

## Type detection

Field columns are detected from up to `sample_size` rows: the first type every non-empty, non-`nullstr`
value fits, in the order `DATE`, `TIMESTAMP`, `BOOLEAN`, `BIGINT`, `DOUBLE`, `VARCHAR` (with
`bulk_fields=true`, `VARCHAR[][]` first).

- `DATE`: ISO `2026-01-14`, or `20260114` when the file header says `DATEFORMAT=yyyymmdd` (years 1900–2199, so
  8-digit codes that are not plausible dates stay `BIGINT`). `TIMESTAMP`: ISO date plus time
  (`2026-01-14 16:30:00`, `2026-01-14T16:30:00Z`); a column mixing dates and timestamps is `TIMESTAMP`.
  `detect_dates=false` turns this off.
- Values with leading zeros (`00123456`) keep a column `VARCHAR`, because in Bloomberg exports those are
  identifiers (CUSIP, SEDOL, ...) whose zeros would otherwise be lost.
- A Bloomberg null marker such as `N.A.` keeps a column `VARCHAR` unless it is declared with `nullstr` or
  `bloomberg_nulls=true`.

Number parsing is locale independent. A value that does not parse as the column type is returned as `NULL`.
Use `types={'col': 'TYPE'}` to fix individual columns.

## Bulk fields

Bloomberg encodes multi-row fields (dividend history, holders, advisers, ...) as one value:
`;2;<rows>;<cols>;` followed by `<type>;<value>;` per cell. With `bulk_fields=true`, a column whose sampled
values all have that shape is returned as `VARCHAR[][]` (a list of rows, each a list of cell values; the
per-cell type codes are dropped). Values that do not decode exactly (wrong cell count) are `NULL`. Force the
decoding for one column with `types={'DVD_HIST': 'VARCHAR[][]'}`.

```sql
SELECT TICKER, h[1] AS kind, strptime(h[2], '%Y%m%d')::DATE AS ex_date, h[3]::DOUBLE AS amount
FROM read_bberg('test/sql/data/misc/bulk.out', bulk_fields=true), unnest(DVD_HIST) AS t(h);
```

## Historical `.out` notes (`hist=true`)

- `DTE` is parsed as `DATE`.
- If the file header contains `DATEFORMAT`, it is used as a format hint for `DTE` parsing.
- Current explicit format support is `yyyymmdd`.
- If date parsing fails for a row, `DTE` is returned as `NULL` for that row.

## Common examples

### 1) Single `.out`

```sql
SELECT *
FROM read_bberg('test/sql/data/year=2026/month=11/sample_bberg.out')
LIMIT 5;
```

### 2) Single `.out.gz`

```sql
SELECT *
FROM read_bberg('test/sql/data/year=2025/month=1/sample_prices.out.gz', empty_as_null=true)
LIMIT 5;
```

### 3) Glob patterns for `.out` and `.out.gz`

```sql
SELECT count(*)
FROM read_bberg('test/sql/data/year=2025/month=1/*.out', empty_as_null=true);

SELECT count(*)
FROM read_bberg('test/sql/data/year=2025/month=1/*.out.gz', empty_as_null=true);
```

### 4) Hive partitioning style query

```sql
SELECT year, month, count(*) AS rows
FROM read_bberg(
	'test/sql/data/year=*/month=*/*.out.gz',
	hive_partitioning=true,
	empty_as_null=true
)
GROUP BY year, month
ORDER BY year, month;
```

### 5) Hive partition filter pushdown style usage

```sql
SELECT *
FROM read_bberg(
	'test/sql/data/year=2026/month=*/sample_bberg.out',
	hive_partitioning=true,
	filename=true
)
WHERE month='11';
```

### 6) Null markers and empty string handling

```sql
SELECT *
FROM read_bberg(
	'test/sql/data/year=2026/month=11/sample_bberg.out',
	nullstr=['N/A', '-', 'null'],
	empty_as_null=true
)
LIMIT 10;
```

### 7) Historical `.out` read

```sql
SELECT DTE, SECURITY_DESC, * EXCLUDE(PARAMETERS, DTE, SECURITY_DESC)
FROM read_bberg(
	'gethistory_prices.out',
	hist=true
)
LIMIT 20;
```

## Multi-file behavior and schema compatibility

When scanning multiple files (glob, directory or list), field declarations from `START-OF-FIELDS` must match
across files, unless `union_by_name=true`.

If declarations differ, `read_bberg` raises:

`read_bberg encountered mismatched field declarations across files`

With `union_by_name=true` the output has the union of all fields (matched case-insensitively, in any order
per file); fields a file does not declare are `NULL` for its rows.

