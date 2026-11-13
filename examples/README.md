# Sample files

| File | Program | Used in |
|---|---|---|
| [`prices.out`](prices.out) | `getdata` | [README](../README.md#example), [read_bberg](../docs/bberg.md) |
| [`history.out`](history.out) | `gethistory` | [read_bberg: historical files](../docs/bberg.md#historical-files) |
| [`dividends.out`](dividends.out) | `getdata` with a bulk field | [read_bberg: bulk fields](../docs/bberg.md#bulk-fields) |
| [`actions.cax`](actions.cax) | `getactions` | [read_bbcax](../docs/bbcax.md) |

To run the documented queries, start DuckDB in the repository root:

```sh
duckdb -unsigned -c "LOAD 'bberg.linux_amd64.duckdb_extension'; SELECT * FROM 'examples/prices.out';"
```

## Disclaimer

These files contain fictitious test data created solely for documentation purposes. Company names, tickers,
identifiers, prices, volumes, dates and corporate-action details are invented and do not refer to any real
issuer, security or market event. Any resemblance to actual entities or data is coincidental. No data
obtained from Bloomberg or any other data vendor is included.
