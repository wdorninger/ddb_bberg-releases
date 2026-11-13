# read_bbcax

*bberg v0.2.1 · [README](../README.md) · [Shared behaviour](usage.md)*

```sql
read_bbcax(path [, option := value, ...])
```

Reads Bloomberg Data License corporate-action files (`.cax`, optionally gzip-compressed) produced by the
`getactions` program. Each data line becomes one row. The fixed part of the line becomes typed columns, and
the action-specific fields are returned either as a map or, when an action type is selected, as typed
columns.

`path` is a file, a directory, a glob, or a list of these. Inputs, remote storage, type detection, NULL
handling and parallelism are shared with `read_bberg` and are described in [usage.md](usage.md).

The examples on this page use [`examples/actions.cax`](../examples/actions.cax). See the
[sample data disclaimer](../README.md#sample-data-disclaimer).

## File layout

`.cax` files have no `START-OF-FIELDS` section. Every data line has 21 fixed fields, followed by
`NUM_PARAMS` `KEY|VALUE` pairs that depend on the action type:

```
SECURITY_DESC|ID_BB_UNIQUE|SECURITY_ID_NUM|ERROR_CODE|ACTION_ID|ACTION_MNEMONIC|ACTION_FLAG|
ISSUER_NAME|ID_TYPE|ID_VALUE|CRNCY|MKT_SECTOR|ID_BB_UNIQUE_STR|ANNOUNCE_DT|EFFECTIVE_DT|ENTRY_DT|
ID_BB_GLOBAL|ID_BB_GLOBAL_COMPANY|TICKER|EXCH_CODE|NUM_PARAMS|KEY|VALUE|KEY|VALUE|...
```

A security without a pending action has a non-zero `ERROR_CODE` (for example `300`) and its line ends after
that field. `ACTION_FLAG` is `N` (new), `U` (updated) or `D` (deleted). Deleted actions usually carry a
single `CP_DELETE_REASON` parameter instead of the full field set.

Common action types include `DVD_CASH`, `DVD_STOCK`, `STOCK_SPLT`, `ACQUIS`, `DIVEST`, `SPIN`,
`RIGHTS_OFFER`, `EQY_OFFER`, `STOCK_BUY`, `SH_HOLDER_MEET`, `LIST`, `DELIST`, `CHG_NAME`, `CHG_TKR`,
`CHG_PAR` and `CHG_ID`. Any action type present in a file can be read.

## Output schema

| Column | Type | Description |
|---|---|---|
| `PARAMETERS` | `MAP(VARCHAR, VARCHAR)` | Header `KEY=VALUE` pairs of the file the row came from |
| `SECURITY_DESC` | `VARCHAR` | Security as requested |
| `ID_BB_UNIQUE` | `BIGINT` | |
| `SECURITY_ID_NUM` | `BIGINT` | |
| `ERROR_CODE` | `BIGINT` | `0` when an action is present |
| `ACTION_ID` | `BIGINT` | Bloomberg action identifier |
| `ACTION_MNEMONIC` | `VARCHAR` | Action type, e.g. `DVD_CASH` |
| `ACTION_FLAG` | `VARCHAR` | `N`, `U` or `D` |
| `ISSUER_NAME` | `VARCHAR` | |
| `ID_TYPE`, `ID_VALUE` | `VARCHAR` | |
| `CRNCY`, `MKT_SECTOR` | `VARCHAR` | |
| `ID_BB_UNIQUE_STR` | `VARCHAR` | |
| `ANNOUNCE_DT`, `EFFECTIVE_DT`, `ENTRY_DT` | `DATE` | Parsed with the file's `DATEFORMAT` (`yyyymmdd`) |
| `ID_BB_GLOBAL`, `ID_BB_GLOBAL_COMPANY` | `VARCHAR` | FIGIs |
| `TICKER`, `EXCH_CODE` | `VARCHAR` | |
| `NUM_PARAMS` | `BIGINT` | Number of `KEY|VALUE` pairs |

Then, depending on the `action` option:

- **`action` not set**: one more column, `CP_PARAMETERS MAP(VARCHAR, VARCHAR)`, with the line's raw
  `KEY|VALUE` pairs. Placeholder lines have an empty map. All action types share this schema.
- **`action` set**: only lines of the selected action type(s) are returned, and every parameter key seen in
  the sample becomes a typed column. Keys that a line does not carry (for example on `D` lines) are `NULL`.

`filename` and hive partition columns follow when requested. The types of all fixed columns can be changed
with `types`.

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `action` | `VARCHAR` or `VARCHAR[]` | none | Return only these action types (case-insensitive) and expand their parameters into typed columns. A list returns the union of the parameter columns. An empty list applies no filter. An action type that does not occur in the data returns zero rows, not an error. |
| `bloomberg_nulls` | `BOOLEAN` | `false` | Treat `N.A.`, `N.S.`, `N.D.`, `N/A`, `#N/A N/A` and `FLD UNKNOWN` as `NULL`. |
| `nullstr` | `VARCHAR` or `VARCHAR[]` | none | Additional values to read as `NULL`. Compared after trimming. Ignored by type detection. |
| `empty_as_null` | `BOOLEAN` | `false` | Read empty `VARCHAR` values as `NULL`. |
| `types` | `STRUCT` | none | Type of individual fixed or parameter columns by name, e.g. `{'CP_GROSS_AMT': 'DECIMAL(18,6)'}`. Unknown names are an error. |
| `auto_detect` | `BOOLEAN` | `true` | Detect parameter column types from a sample. |
| `all_varchar` | `BOOLEAN` | `false` | Read all parameter columns as `VARCHAR`. |
| `detect_dates` | `BOOLEAN` | `true` | Detect `DATE` and `TIMESTAMP` parameter columns. |
| `bulk_fields` | `BOOLEAN` | `false` | Decode bulk parameters such as `CP_ADVISORS` into `VARCHAR[][]`. Use together with `bloomberg_nulls`, because `N.A.` values would otherwise keep the column `VARCHAR`. |
| `sample_size` | `BIGINT` | `20480` | Lines of the selected action types sampled for column discovery and type detection. `0` or less samples all. |
| `filename` | `BOOLEAN` | `false` | Add a `filename` column. |
| `hive_partitioning` | `BOOLEAN` | `false` | Add a `VARCHAR` column for each `key=value` directory in the path. |
| `encoding` | `VARCHAR` | `'utf-8'` | `'utf-8'` (validated) or `'latin-1'`. |
| `ignore_errors` | `BOOLEAN` | `false` | Skip data lines with fewer than four fields and lines that are not valid in the chosen encoding. |
| `parallel` | `BOOLEAN` | `true` | Scan on several threads. `false` reads on one thread in input order. |
| `split_size` | `BIGINT` | `16777216` | Byte size of the ranges a large uncompressed file is split into. |

## Examples

### All actions, generic shape

```sql
SELECT SECURITY_DESC, ERROR_CODE, ACTION_MNEMONIC, ACTION_FLAG, EFFECTIVE_DT, CP_PARAMETERS
FROM read_bbcax('examples/actions.cax', parallel = false);
```

| SECURITY_DESC | ERROR_CODE | ACTION_MNEMONIC | ACTION_FLAG | EFFECTIVE_DT | CP_PARAMETERS |
|---|---:|---|---|---|---|
| QKW US Equity | 0 | DVD_CASH | N | 2026-02-13 | `{CP_RECORD_DT=20260213, CP_PAY_DT=20260227, CP_GROSS_AMT=0.54, CP_DVD_CRNCY=USD, CP_FREQ=4, CP_DVD_TYP=1000}` |
| PDL US Equity | 300 | NULL | NULL | NULL | `{}` |
| MRSH LN Equity | 0 | DVD_CASH | U | 2026-01-22 | `{CP_RECORD_DT=20260123, CP_PAY_DT=20260220, CP_GROSS_AMT=31.25, CP_DVD_CRNCY=GBp, CP_FREQ=2, CP_DVD_TYP=N.A.}` |
| FTHR GY Equity | 0 | STOCK_SPLT | N | 2026-03-02 | `{CP_RATIO=0.25, CP_TERMS=4 for 1, CP_RECORD_DT=20260303}` |
| WDDL FP Equity | 0 | ACQUIS | N | NULL | `{CP_ACQUIRER_NAME=QUACKWORKS INC, CP_TKR=QKW US, CP_CASH=18.50, ...}` |
| PDL US Equity | 0 | DVD_CASH | D | 2025-12-15 | `{CP_DELETE_REASON=Announced in error}` |

Values of the map are always text. Extract a single value with `CP_PARAMETERS['CP_GROSS_AMT']`.

### Count actions by type

```sql
SELECT ACTION_MNEMONIC, count(*) AS actions
FROM 'examples/actions.cax'
WHERE ERROR_CODE = 0
GROUP BY ALL
ORDER BY ALL;
```

| ACTION_MNEMONIC | actions |
|---|---:|
| ACQUIS | 1 |
| DVD_CASH | 3 |
| STOCK_SPLT | 1 |

### One action type as typed columns

```sql
DESCRIBE SELECT * FROM read_bbcax('examples/actions.cax', action = 'DVD_CASH', bloomberg_nulls = true);
```

The fixed columns are followed by:

| column_name | column_type |
|---|---|
| CP_RECORD_DT | DATE |
| CP_PAY_DT | DATE |
| CP_GROSS_AMT | DOUBLE |
| CP_DVD_CRNCY | VARCHAR |
| CP_FREQ | BIGINT |
| CP_DVD_TYP | BIGINT |
| CP_DELETE_REASON | VARCHAR |

```sql
SELECT SECURITY_DESC, ACTION_FLAG, CP_RECORD_DT, CP_PAY_DT, CP_GROSS_AMT, CP_DVD_CRNCY
FROM read_bbcax('examples/actions.cax', action = 'DVD_CASH', bloomberg_nulls = true,
                types = {'CP_GROSS_AMT': 'DECIMAL(18,6)'}, parallel = false);
```

| SECURITY_DESC | ACTION_FLAG | CP_RECORD_DT | CP_PAY_DT | CP_GROSS_AMT | CP_DVD_CRNCY |
|---|---|---|---|---:|---|
| QKW US Equity | N | 2026-02-13 | 2026-02-27 | 0.540000 | USD |
| MRSH LN Equity | U | 2026-01-23 | 2026-02-20 | 31.250000 | GBp |
| PDL US Equity | D | NULL | NULL | NULL | NULL |

### Deleted actions

```sql
SELECT SECURITY_DESC, ACTION_ID, CP_DELETE_REASON
FROM read_bbcax('examples/actions.cax', action = 'DVD_CASH')
WHERE ACTION_FLAG = 'D';
```

| SECURITY_DESC | ACTION_ID | CP_DELETE_REASON |
|---|---:|---|
| PDL US Equity | 900000099 | Announced in error |

### Several action types

```sql
SELECT SECURITY_DESC, ACTION_MNEMONIC, CP_RECORD_DT, CP_GROSS_AMT, CP_RATIO
FROM read_bbcax('examples/actions.cax', action = ['DVD_CASH', 'STOCK_SPLT'],
                bloomberg_nulls = true, parallel = false)
WHERE ACTION_FLAG <> 'D';
```

| SECURITY_DESC | ACTION_MNEMONIC | CP_RECORD_DT | CP_GROSS_AMT | CP_RATIO |
|---|---|---|---:|---:|
| QKW US Equity | DVD_CASH | 2026-02-13 | 0.54 | NULL |
| MRSH LN Equity | DVD_CASH | 2026-01-23 | 31.25 | NULL |
| FTHR GY Equity | STOCK_SPLT | 2026-03-03 | NULL | 0.25 |

### Bulk parameters

```sql
SELECT SECURITY_DESC, CP_ACQUIRER_NAME, CP_CASH, a[1] AS role, a[2] AS adviser
FROM read_bbcax('examples/actions.cax', action = 'ACQUIS', bulk_fields = true, bloomberg_nulls = true),
     unnest(CP_ADVISORS) AS t(a);
```

| SECURITY_DESC | CP_ACQUIRER_NAME | CP_CASH | role | adviser |
|---|---|---:|---|---|
| WDDL FP Equity | QUACKWORKS INC | 18.5 | Acquirer financial adviser | Example Partners LLP |
| WDDL FP Equity | QUACKWORKS INC | 18.5 | Target legal adviser | Sample & Co |

### Upcoming payments across a daily archive

```sql
SELECT ISSUER_NAME, CP_PAY_DT, CP_GROSS_AMT, CP_DVD_CRNCY
FROM read_bbcax('archive/cax/*.cax.gz', action = 'DVD_CASH', bloomberg_nulls = true)
WHERE ACTION_FLAG <> 'D' AND CP_PAY_DT BETWEEN DATE '2026-02-01' AND DATE '2026-02-28'
ORDER BY CP_PAY_DT;
```

## Multiple files

With `action` set, the parameter columns are discovered across the sampled lines of all files. A key that
appears in one file but not in another is `NULL` where it is missing. No `union_by_name` option is needed.
`PARAMETERS` and the `DATEFORMAT` used for the date columns are taken from each row's own file.
