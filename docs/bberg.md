# read_bberg

*bberg v0.2.1 · [README](../README.md) · [Shared behaviour](usage.md)*

```sql
read_bberg(path [, option := value, ...])
```

Reads Bloomberg Data License request/reply files (`.out`, optionally gzip-compressed) produced by the
`getdata` and `gethistory` programs. Each data line becomes one row, and each field listed in the file's
`START-OF-FIELDS` section becomes one column.

`path` is a file, a directory, a glob, or a list of these. Inputs, remote storage, type detection, NULL
handling and parallelism are shared with `read_bbcax` and are described in [usage.md](usage.md).

The examples on this page use the sample files in [`examples/`](../examples/). See the
[sample data disclaimer](../README.md#sample-data-disclaimer).

## File layout

A reply file has a header of `KEY=VALUE` lines, the list of requested fields and the data section:

```
START-OF-FILE
RUNDATE=20260114
PROGRAMNAME=getdata
DATEFORMAT=yyyymmdd
START-OF-FIELDS
TICKER
NAME
PX_LAST
END-OF-FIELDS
START-OF-DATA
QKW US Equity|0|3|QKW|QUACKWORKS INC|184.32|
PDL US Equity|0|3|PDL|PUDDLE LOGISTICS CORP|47.05|
END-OF-DATA
END-OF-FILE
```

Every data line starts with the security, an error code and the number of fields, followed by one value per
requested field. In `gethistory` files a date follows the field count. Lines that begin with
`comment_prefix` (default `#`) are skipped.

## Output schema

| Column | Type | Description |
|---|---|---|
| `PARAMETERS` | `MAP(VARCHAR, VARCHAR)` | Header `KEY=VALUE` pairs of the file the row came from (`RUNDATE`, `PROGRAMNAME`, `DATEFORMAT`, ...) |
| `SECURITY_DESC` | `VARCHAR` | Security as requested, e.g. `QKW US Equity` |
| `ERROR_CODE` | `BIGINT` | Bloomberg return code for the security; `0` means success |
| `NUM_FIELDS` | `BIGINT` | Number of field values on the line |
| `DTE` | `DATE` | Only with `hist = true`: the observation date |
| *one column per field* | detected | Named after the field mnemonic, typed by [type detection](usage.md#type-detection) |
| `filename` | `VARCHAR` | Only with `filename = true` |
| *partition keys* | `VARCHAR` | Only with `hive_partitioning = true` |

`DESCRIBE` shows the resolved schema for a file:

```sql
DESCRIBE SELECT * FROM read_bberg('examples/prices.out', bloomberg_nulls = true);
```

| column_name | column_type |
|---|---|
| PARAMETERS | MAP(VARCHAR, VARCHAR) |
| SECURITY_DESC | VARCHAR |
| ERROR_CODE | BIGINT |
| NUM_FIELDS | BIGINT |
| TICKER | VARCHAR |
| NAME | VARCHAR |
| CRNCY | VARCHAR |
| PX_LAST | DOUBLE |
| PX_VOLUME | BIGINT |
| EQY_DVD_YLD_IND | DOUBLE |
| LAST_UPDATE_DT | DATE |
| ID_CUSIP | VARCHAR |

`ID_CUSIP` stays `VARCHAR` because its values have leading zeros. Without `bloomberg_nulls = true`, the
`N.A.` placeholders in the sample file would make every field column `VARCHAR`.

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `hist` | `BOOLEAN` | `false` | Read the `gethistory` layout. Adds the `DTE` column after `NUM_FIELDS`. |
| `bloomberg_nulls` | `BOOLEAN` | `false` | Treat Bloomberg's placeholders `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A` and `FLD UNKNOWN` as `NULL`. |
| `nullstr` | `VARCHAR` or `VARCHAR[]` | none | Additional values to read as `NULL`. Compared after trimming. Ignored by type detection. |
| `empty_as_null` | `BOOLEAN` | `false` | Read empty `VARCHAR` values as `NULL`. |
| `types` | `STRUCT` | none | Type of individual columns by name, e.g. `{'PX_LAST': 'DECIMAL(18,6)'}`. Other columns keep detection. Case-insensitive; an unknown name is an error. Takes precedence over `all_varchar`. Cannot be combined with `columns`. |
| `columns` | `STRUCT` | none | Full schema override: every data column (`SECURITY_DESC`, `ERROR_CODE`, `NUM_FIELDS`, `DTE` if `hist`, then all fields), in order, with name and type. A different column count is an error. |
| `auto_detect` | `BOOLEAN` | `true` | Detect field types from a sample. `false` reads all field columns as `VARCHAR`. |
| `all_varchar` | `BOOLEAN` | `false` | Read all field columns as `VARCHAR`. The fixed columns keep their types. |
| `detect_dates` | `BOOLEAN` | `true` | Detect `DATE` and `TIMESTAMP` columns. With `false`, such columns become `BIGINT` or `VARCHAR`. |
| `bulk_fields` | `BOOLEAN` | `false` | Decode bulk fields into `VARCHAR[][]`. See [Bulk fields](#bulk-fields). |
| `sample_size` | `BIGINT` | `20480` | Rows sampled for type detection. `0` or less samples all rows. |
| `union_by_name` | `BOOLEAN` | `false` | Combine files with different field lists. Fields are matched by name, case-insensitively; missing fields are `NULL`. Without it, differing field lists are an error. |
| `filename` | `BOOLEAN` | `false` | Add a `filename` column. |
| `hive_partitioning` | `BOOLEAN` | `false` | Add a `VARCHAR` column for each `key=value` directory in the path. |
| `comment_prefix` | `VARCHAR` | `'#'` | Skip lines starting with this prefix in the fields and data sections. |
| `encoding` | `VARCHAR` | `'utf-8'` | `'utf-8'` (validated) or `'latin-1'` (aliases `latin1`, `iso-8859-1`). |
| `ignore_errors` | `BOOLEAN` | `false` | Skip data lines with fewer than three values and lines that are not valid in the chosen encoding. |
| `parallel` | `BOOLEAN` | `true` | Scan files and byte ranges of large files on several threads. `false` reads on one thread in input order. |
| `split_size` | `BIGINT` | `16777216` | Byte size of the ranges a large uncompressed file is split into. |
| `store_rejects` | `BOOLEAN` | `false` | Accepted for compatibility with `read_csv`. `true` is an error; use `ignore_errors`. |
| `rejects_scan` | `VARCHAR` | none | Accepted for compatibility with `read_csv`. Setting it is an error. |

Supported type names for `types` and `columns` are listed in [usage.md](usage.md#explicit-types).

## Historical files

`gethistory` files carry one line per security and date. Read them with `hist = true`:

```sql
SELECT SECURITY_DESC, DTE, PX_OPEN, PX_LAST, PX_VOLUME
FROM read_bberg('examples/history.out', hist = true)
ORDER BY SECURITY_DESC, DTE;
```

| SECURITY_DESC | DTE | PX_OPEN | PX_LAST | PX_VOLUME |
|---|---|---:|---:|---:|
| PDL US Equity | 2026-01-12 | 46.2 | 46.71 | 902400 |
| PDL US Equity | 2026-01-13 | 46.7 | 47.12 | 811700 |
| PDL US Equity | 2026-01-14 | 47.1 | 47.05 | 880150 |
| QKW US Equity | 2026-01-12 | 181.1 | 183.2 | 2104300 |
| QKW US Equity | 2026-01-13 | 183.25 | 185.02 | 2488100 |
| QKW US Equity | 2026-01-14 | 185.0 | 184.32 | 2315400 |

`DTE` is parsed using the file's `DATEFORMAT` header (`yyyymmdd` is supported) or as an ISO date. A date that
cannot be parsed is `NULL`.

## Bulk fields

Bloomberg encodes multi-row fields, such as dividend history, as a single value: `;2;<rows>;<cols>;`
followed by a `<type>;<value>;` pair for every cell. With `bulk_fields = true`, a column whose values all have
this shape is returned as `VARCHAR[][]`: a list of rows, each a list of cell values. Values that do not
decode are `NULL`. To decode one column regardless of detection, use `types = {'DVD_HIST': 'VARCHAR[][]'}`.

```sql
SELECT TICKER,
       h[1]                             AS dividend_type,
       strptime(h[2], '%Y%m%d')::DATE   AS ex_date,
       h[3]::DOUBLE                     AS amount
FROM read_bberg('examples/dividends.out', bulk_fields = true),
     unnest(DVD_HIST) AS t(h);
```

| TICKER | dividend_type | ex_date | amount |
|---|---|---|---:|
| QKW | Regular Cash | 2025-11-14 | 0.52 |
| QKW | Regular Cash | 2025-08-15 | 0.5 |
| PDL | Special Cash | 2025-12-01 | 1.6 |

## Examples

### Successful securities only, with failures listed separately

```sql
SELECT SECURITY_DESC, ERROR_CODE, NAME, PX_LAST
FROM read_bberg('examples/prices.out', bloomberg_nulls = true)
ORDER BY ERROR_CODE DESC, SECURITY_DESC;
```

| SECURITY_DESC | ERROR_CODE | NAME | PX_LAST |
|---|---:|---|---:|
| XYZQ US Equity | 10 | NULL | NULL |
| FTHR GY Equity | 0 | FEDERWERK AG | 92.1 |
| MRSH LN Equity | 0 | MARSHLAND UTILITIES PLC | 1268.5 |
| PDL US Equity | 0 | PUDDLE LOGISTICS CORP | 47.05 |
| QKW US Equity | 0 | QUACKWORKS INC | 184.32 |
| WDDL FP Equity | 0 | WADDLE SA | 15.884 |

### Exact decimals

```sql
SELECT TICKER, PX_LAST
FROM read_bberg('examples/prices.out', bloomberg_nulls = true,
                types = {'PX_LAST': 'DECIMAL(18,4)'})
WHERE ERROR_CODE = 0;
```

### Header values

```sql
SELECT DISTINCT PARAMETERS['PROGRAMNAME'] AS program, PARAMETERS['RUNDATE'] AS run_date
FROM read_bberg('examples/prices.out');
```

| program | run_date |
|---|---|
| getdata | 20260114 |

### Several files with different field lists

```sql
SELECT filename, count(*) AS rows
FROM read_bberg(['examples/prices.out', 'examples/history.out'],
                union_by_name = true, filename = true)
GROUP BY filename
ORDER BY filename;
```

| filename | rows |
|---|---:|
| examples/history.out | 6 |
| examples/prices.out | 6 |

### Partitioned archive

For a directory tree such as `archive/year=2026/month=01/getdata_20260114.out.gz`:

```sql
SELECT year, month, count(*) AS rows
FROM read_bberg('archive/year=*/month=*/*.out.gz', hive_partitioning = true, bloomberg_nulls = true)
GROUP BY ALL
ORDER BY ALL;
```

### Loading into a table

```sql
SET preserve_insertion_order = false;   -- allows a parallel load
CREATE TABLE prices AS
SELECT * EXCLUDE (PARAMETERS)
FROM read_bberg('archive/**/*.out.gz', union_by_name = true, bloomberg_nulls = true);
```

### Replacement scan

```sql
SELECT count(*) FROM 'examples/prices.out';
```

| count_star() |
|---:|
| 6 |

## Multiple files

All files must declare the same fields in `START-OF-FIELDS`, unless `union_by_name = true`. Otherwise the
query fails with `read_bberg encountered mismatched field declarations across files`. With
`union_by_name = true`, the output contains the union of all fields, and fields that a file does not declare
are `NULL` for its rows.
