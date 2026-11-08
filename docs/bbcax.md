# `read_bbcax`

*Documentation for `bberg` v0.2.0. [Back to the README](../README.md).*

> Paths such as `test/sql/data/...` in the examples refer to the extension's own test files, which are not published. Use your own files.

## Overview

`read_bbcax` is a table function for Bloomberg corporate-actions bulk export files (the
`getactions`-program `.cax` files, plain text or gzip-compressed). Unlike `.out` files, `.cax` files
have no `START-OF-FIELDS` section — most rows are placeholders ("no pending action for this
security"), and the rows that do carry an action encode their fields as self-describing `KEY|VALUE`
pairs at the end of the line, and the key set differs by corporate-action type (`ACQUIS`, `DVD_CASH`,
`STOCK_BUY`, `SH_HOLDER_MEET`, `DIVEST`, `DELIST`, `LIST`, `STOCK_SPLT`, `RIGHTS_OFFER`, `EQY_OFFER`,
`SPIN`, `DVD_STOCK`, `CHG_NAME`, `CHG_TKR`, `CHG_PAR`, `CHG_ID`, and any others present in the file).

Behaviour shared with the other readers (inputs, remote files, types, NULLs, parallelism) is described in
[usage.md](usage.md).

## Quick start

```sql
LOAD '/path/to/bberg.linux_amd64.duckdb_extension';

-- All rows (placeholders + actions), with action-specific fields as a generic map
SELECT *
FROM read_bbcax('test/sql/data/cax/mini_actions.cax')
LIMIT 5;

-- Only ACQUIS rows, with ACQUIS's own fields exploded into typed columns
SELECT *
FROM read_bbcax('test/sql/data/cax/mini_actions.cax', action='ACQUIS');
```

## `.cax` row layout

```
SECURITY_DESC|ID_BB_UNIQUE|SECURITY_ID_NUM|ERROR_CODE|ACTION_ID|ACTION_MNEMONIC|ACTION_FLAG|
ISSUER_NAME|ID_TYPE|ID_VALUE|CRNCY|MKT_SECTOR|ID_BB_UNIQUE_STR|ANNOUNCE_DT|EFFECTIVE_DT|ENTRY_DT|
ID_BB_GLOBAL|ID_BB_GLOBAL_COMPANY|TICKER|EXCH_CODE|NUM_PARAMS|<KEY|VALUE pairs, NUM_PARAMS of them>
```

A row with no pending corporate action for the security stops after `ERROR_CODE` (a non-zero code,
e.g. `300`); `ACTION_ID` through `NUM_PARAMS` and the trailing `KEY|VALUE` pairs are only present once
an action exists (`ERROR_CODE = 0`). `ACTION_FLAG` is `N` (new), `U` (updated), or `D` (deleted); `D`
rows typically carry only a single `CP_DELETE_REASON` parameter instead of the action's full field set.

## Output schema

`read_bbcax` always includes:

- `PARAMETERS MAP(VARCHAR, VARCHAR)` — the file header's `KEY=VALUE` pairs (e.g. `PROGRAMNAME`,
  `DATEFORMAT`, `TIMESTARTED`), same as `read_bberg`.
- The 21 fixed columns from the row layout above (`SECURITY_DESC` … `NUM_PARAMS`).
  `ANNOUNCE_DT`, `EFFECTIVE_DT`, and `ENTRY_DT` are parsed as `DATE` using the file header's
  `DATEFORMAT` (currently only `yyyymmdd` is implemented). Use `types=` to change any of these types.

Then, depending on whether `action` is set:

- **`action` unset (default)**: a trailing `CP_PARAMETERS MAP(VARCHAR, VARCHAR)` column holds that
  row's raw `KEY|VALUE` pairs (empty map for placeholder rows). Every action type's rows are returned,
  in one uniform schema.
- **`action` set**: only rows whose `ACTION_MNEMONIC` matches (case-insensitively) are returned, and
  one typed column per parameter key seen for that action is appended (superset across sampled rows,
  so e.g. `ACQUIS`'s `D`-flag rows and full `N`/`U`-flag rows share one schema — columns absent on a
  given row are `NULL`). Types are inferred the same way as `read_bberg`'s dynamic field columns, so
  `CP_*_DT` parameters are `DATE` in `DATEFORMAT=yyyymmdd` files. `action` can also be a list
  (`action=['DVD_CASH', 'DVD_STOCK']`): rows of any listed action are returned, with the union of their
  parameter columns. An empty list means no filter.

Optional extra output columns:

- `filename VARCHAR` when `filename=true`
- one `VARCHAR` column per detected hive partition key when `hive_partitioning=true`

## `read_bbcax` options

| Option | Type | Default | Behavior |
|---|---|---|---|
| `action` | `VARCHAR` or `LIST<VARCHAR>` | unset | Filters to rows with this `ACTION_MNEMONIC` (or any of the listed ones, case-insensitive) and explodes their parameters into typed columns. Unset returns all rows with a generic `CP_PARAMETERS` map. An action that never occurs in the file returns zero rows (not an error). |
| `types` | `STRUCT` | unset | Override the type of individual fixed or action columns by name, e.g. `types={'CP_GROSS_AMT': 'DECIMAL(18,6)'}`. Unknown names are an error. Type names as in `read_bberg`'s `columns`. |
| `detect_dates` | `BOOLEAN` | `true` | Detect `DATE` / `TIMESTAMP` action columns. `false` keeps them `BIGINT` / `VARCHAR`. |
| `bulk_fields` | `BOOLEAN` | `false` | Decode bulk action parameters (e.g. `CP_ADVISORS`, `;2;rows;cols;...`) into `VARCHAR[][]`. Combine with `bloomberg_nulls=true`, because `N.A.` placeholders otherwise keep the column `VARCHAR`. |
| `encoding` | `VARCHAR` | `'utf-8'` | `'utf-8'` validates text; `'latin-1'` converts ISO-8859-1 files. |
| `split_size` | `BIGINT` | `16777216` | Byte size of the ranges a large uncompressed file is split into for parallel scanning. |
| `nullstr` | `VARCHAR` or `LIST<VARCHAR>` | unset | Same as `read_bberg`/`read_csv`: any trimmed value equal to one of these markers is returned as `NULL`. Useful for Bloomberg's `N.A.` marker, e.g. `nullstr='N.A.'`. |
| `empty_as_null` | `BOOLEAN` | `false` | Converts empty strings to `NULL`. |
| `all_varchar` | `BOOLEAN` | `false` | Forces `action`-specific columns to `VARCHAR` instead of inferred types. |
| `auto_detect` | `BOOLEAN` | `true` | Infer `action`-specific column types from sampled rows. |
| `sample_size` | `BIGINT` | `20480` | Number of action-matching rows sampled for column discovery/type inference when `action` is set. Values `<= 0` mean "sample all matching rows". |
| `filename` | `BOOLEAN` | `false` | Appends a `filename` output column. |
| `hive_partitioning` | `BOOLEAN` | `false` | Extract path segments `key=value` into output columns (as `VARCHAR`). |
| `ignore_errors` | `BOOLEAN` | `false` | Skips malformed data rows with fewer than 4 columns and lines that are not valid UTF-8; otherwise the query errors out. |
| `bloomberg_nulls` | `BOOLEAN` | `false` | Adds Bloomberg's "no value" markers to `nullstr`: `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A`, `FLD UNKNOWN`. |
| `parallel` | `BOOLEAN` | `true` | Scan files, and byte ranges of large uncompressed files, on multiple threads (row order is then unspecified; `false` keeps file order). |

`path` may be a file, directory, glob or a `LIST` of them, and a bare `.cax` / `.cax.gz` path in `FROM` is read
with `read_bbcax` automatically (`SELECT * FROM 'actions.cax'`). Remote paths (`s3://`, `https://`, ... via
`httpfs`) work as explicit file paths. `PARAMETERS` and the `DATEFORMAT` used for the `*_DT` columns come from
each row's own file. Action-parameter values with leading zeros keep their column `VARCHAR`, and `nullstr`
markers are ignored by type detection.

## Examples

The examples read `getactions.cax`, standing for any getactions export; substitute your own `.cax` file.

### 1) All corporate actions of one type, with their own columns

```sql
SELECT SECURITY_DESC, ACTION_FLAG, CP_TKR, CP_TOT_AMT, CP_STAT
FROM read_bbcax('getactions.cax', action='ACQUIS')
WHERE ACTION_FLAG != 'D';
```

### 2) Every action in the file, generic shape

```sql
SELECT SECURITY_DESC, ACTION_MNEMONIC, CP_PARAMETERS
FROM read_bbcax('getactions.cax')
WHERE ERROR_CODE = 0;
```

### 3) Null markers

```sql
SELECT *
FROM read_bbcax('getactions.cax', action='DVD_CASH', nullstr='N.A.');
```

### 4) Several actions, exact amounts

```sql
SELECT SECURITY_DESC, ACTION_MNEMONIC, CP_RECORD_DT, CP_GROSS_AMT
FROM read_bbcax('getactions.cax', action=['DVD_CASH', 'DVD_STOCK'],
                bloomberg_nulls=true, types={'CP_GROSS_AMT': 'DECIMAL(18,6)'});
```

### 5) Advisers of each acquisition (bulk field)

```sql
SELECT SECURITY_DESC, a[1] AS role, a[2] AS adviser
FROM read_bbcax('getactions.cax', action='ACQUIS', bulk_fields=true,
                bloomberg_nulls=true), unnest(CP_ADVISORS) AS t(a);
```

## Multi-file behavior

When scanning multiple files (glob, directory or list) with `action` set, the parameter-key superset (and
therefore the output schema) is built across all matched files' sampled rows — a key present in one
file but not another is simply `NULL` on rows where it doesn't appear, no `union_by_name` flag needed.
