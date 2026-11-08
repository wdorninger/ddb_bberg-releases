# bberg — DuckDB extension (binaries)

Prebuilt binaries of the `bberg` DuckDB extension, which adds `read_bberg` and `read_bbcax` for reading
Bloomberg Data License `.out` (getdata/gethistory) and corporate-actions `.cax` (getactions) files with SQL.

This repository contains **only the compiled extension**; the source code is not public.
Current version: **v0.2.0**. Download the files from the [Releases](https://github.com/wdorninger/ddb_bberg-releases/releases) page.

## Files

| File | Platform |
|---|---|
| `bberg.linux_amd64.duckdb_extension` | Linux x86-64 (glibc ≥ 2.28) |
| `bberg.windows_amd64.duckdb_extension` | Windows x86-64 |
| `bberg.duckdb_extension.wasm` | DuckDB-Wasm (`wasm_eh`, browser / Node) |
| `SHA256SUMS` | checksums |

## DuckDB version compatibility

Works with **DuckDB ≥ 1.5.6, including 2.0.x**. The extension is built on DuckDB's stable C extension API (version
v1.5.6), so the same binary loads into every newer DuckDB release without rebuilding. Remote storage (httpfs/S3),
secrets and caching come from the DuckDB that loads it.

## Usage

The extension is not signed by DuckDB, so unsigned extensions must be allowed:

```sh
duckdb -unsigned
```

```sql
LOAD '/path/to/bberg.linux_amd64.duckdb_extension';
SELECT * FROM read_bberg('prices.out');
SELECT * FROM read_bbcax('actions.cax', action='DVD_CASH');
```

Python: `duckdb.connect(config={"allow_unsigned_extensions": "true"})`, then the same `LOAD`.

DuckDB-Wasm: open the database with `allowUnsignedExtensions: true` and
`LOAD 'https://<your host>/bberg.duckdb_extension.wasm'` (served with CORS). Globs are not available in the
Wasm build; pass explicit file paths or a list.

Keep the file name starting with `bberg.`: DuckDB derives the extension name from the part before the first dot.

## Documentation

Documentation for v0.2.0:

| Page | Contents |
|---|---|
| [docs/bberg.md](docs/bberg.md) | `read_bberg`: `.out` layout, schema, every option, `hist=true`, examples |
| [docs/bbcax.md](docs/bbcax.md) | `read_bbcax`: `.cax` layout, `action=`, options, examples |
| [docs/usage.md](docs/usage.md) | Inputs, globs, remote files, replacement scan, types, NULLs, encodings, parallelism, errors |
| [docs/wasm.md](docs/wasm.md) | Using the DuckDB-Wasm build in a browser or Node |

## Third-party notices

See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
