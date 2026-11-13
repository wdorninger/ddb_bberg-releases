# Java (JDBC) integration

*bberg v0.2.1 · [README](../README.md)*

This page describes how to use `bberg` from Java through the DuckDB JDBC driver. The short version is in the
[README](../README.md#using-bberg-with-the-duckdb-jdbc-driver).

## Requirements

- DuckDB JDBC driver `org.duckdb:duckdb_jdbc` 1.5.6.0 or newer (2.0.x is supported).
- The `bberg` binary that matches the JVM's operating system and architecture (see the table under
  [Choosing the binary at runtime](#choosing-the-binary-at-runtime)).
- Unsigned extensions allowed on the database, as described below.

## Allowing unsigned extensions

DuckDB verifies the signature of every extension it loads. `bberg` is distributed outside DuckDB's extension
repository and is not signed with DuckDB's key, so signature checking must be switched off for the database
that loads it. This is done with the `allow_unsigned_extensions` configuration option.

The option is read only when the database starts. Pass it as a connection property:

```java
Properties props = new Properties();
props.setProperty("allow_unsigned_extensions", "true");
Connection conn = DriverManager.getConnection("jdbc:duckdb:", props);
```

Setting it later does not work:

```
SET allow_unsigned_extensions = true;
-- Invalid Input Error: Cannot change allow_unsigned_extensions setting while database is running
```

Allowing unsigned extensions permits `LOAD` of any extension file the process can read. Keep the extension
directory writable only by trusted accounts, and verify downloaded binaries against `SHA256SUMS`.

## In-memory databases

The JDBC URL `jdbc:duckdb:` opens a new, private in-memory database for every
`DriverManager.getConnection` call. Extensions are loaded per database, so:

- Run `LOAD` on each connection obtained from `DriverManager`.
- Connections created with `DuckDBConnection.duplicate()` belong to the same database. They see the loaded
  extension and any tables created through the original connection.

```java
import org.duckdb.DuckDBConnection;
import java.sql.*;
import java.util.Properties;

public final class BbergExample {
    public static void main(String[] args) throws SQLException {
        Properties props = new Properties();
        props.setProperty("allow_unsigned_extensions", "true");

        try (DuckDBConnection conn =
                 (DuckDBConnection) DriverManager.getConnection("jdbc:duckdb:", props)) {

            try (Statement stmt = conn.createStatement()) {
                stmt.execute("LOAD '/opt/duckdb/extensions/bberg.linux_amd64.duckdb_extension'");
                stmt.execute("""
                    CREATE TABLE prices AS
                    SELECT SECURITY_DESC, TICKER, PX_LAST, LAST_UPDATE_DT
                    FROM read_bberg('/data/bbg/prices.out', bloomberg_nulls = true)
                    WHERE ERROR_CODE = 0
                    """);
            }

            // A second connection to the same in-memory database, e.g. for another thread.
            try (Connection worker = conn.duplicate();
                 PreparedStatement ps = worker.prepareStatement(
                     "SELECT TICKER, PX_LAST FROM prices WHERE PX_LAST > ? ORDER BY TICKER")) {
                ps.setDouble(1, 50.0);
                try (ResultSet rs = ps.executeQuery()) {
                    while (rs.next()) {
                        System.out.printf("%-6s %10.3f%n", rs.getString(1), rs.getDouble(2));
                    }
                }
            }
        }
    }
}
```

Output for the sample file [`examples/prices.out`](../examples/prices.out):

```
FTHR       92.100
MRSH     1268.500
QKW       184.320
```

## File-based databases

The same property applies to a database file, for example `jdbc:duckdb:/var/lib/app/market.duckdb`. Within one
JVM, all connections to the same file share a single database instance, so the extension needs to be loaded
only once. Every connection to that file must be opened with the same properties; a connection without
`allow_unsigned_extensions` fails with
`Can't open a connection to same database file with a different configuration than existing connections`.

Running `LOAD` again for an extension that is already loaded is harmless, so it is safe to load on every
connection.

## Connection pools

With a pool such as HikariCP, pass the property through the data source properties and load the extension
in the connection-init SQL, so every pooled connection is ready to use:

```java
HikariConfig cfg = new HikariConfig();
cfg.setJdbcUrl("jdbc:duckdb:");
cfg.addDataSourceProperty("allow_unsigned_extensions", "true");
cfg.setConnectionInitSql("LOAD '/opt/duckdb/extensions/bberg.linux_amd64.duckdb_extension'");
HikariDataSource ds = new HikariDataSource(cfg);
```

Spring Boot equivalent (`application.properties`):

```properties
spring.datasource.url=jdbc:duckdb:
spring.datasource.hikari.data-source-properties.allow_unsigned_extensions=true
spring.datasource.hikari.connection-init-sql=LOAD '/opt/duckdb/extensions/bberg.linux_amd64.duckdb_extension'
```

With `jdbc:duckdb:` each pooled connection is its own in-memory database, so tables created on one pooled
connection are not visible on another. Use a database file if the data must be shared across the pool.

## Choosing the binary at runtime

`LOAD` needs a path on the file system. When the application runs on several platforms, ship all binaries
and select one from the JVM's system properties:

| `os.name` | `os.arch` | File |
|---|---|---|
| Linux | `amd64` | `bberg.linux_amd64.duckdb_extension` |
| Mac OS X | `aarch64` | `bberg.osx_arm64.duckdb_extension` |
| Mac OS X | `x86_64` | `bberg.osx_amd64.duckdb_extension` |
| Windows | `amd64` | `bberg.windows_amd64.duckdb_extension` |

If the binaries are packaged inside the application JAR, copy the selected one to a temporary directory first.
Keep the original file name, because DuckDB takes the extension name from the part before the first dot:

```java
static Path extractExtension(String resourceName) throws IOException {
    Path dir = Files.createTempDirectory("duckdb-ext");
    Path target = dir.resolve(resourceName);               // e.g. bberg.linux_amd64.duckdb_extension
    try (InputStream in = BbergExample.class.getResourceAsStream("/duckdb/" + resourceName)) {
        if (in == null) throw new FileNotFoundException(resourceName);
        Files.copy(in, target);
    }
    target.toFile().deleteOnExit();
    dir.toFile().deleteOnExit();
    return target;
}

// stmt.execute("LOAD '" + extractExtension(name).toString().replace("'", "''") + "'");
```

On Windows, paths with backslashes work as they are inside the SQL string literal.

## Troubleshooting

| Message | Cause and fix |
|---|---|
| `... could not be loaded because its signature is either missing or invalid and unsigned extensions are disabled by configuration (allow_unsigned_extensions)` | The property was not passed when the database was opened. Add it to the connection `Properties` (or the pool's data source properties). |
| `Cannot change allow_unsigned_extensions setting while database is running` | The setting was issued with `SET`. Pass it as a connection property instead. |
| `Catalog Error: Table Function with name read_bberg does not exist!` | The connection belongs to a database on which `LOAD` was not run. See [In-memory databases](#in-memory-databases). |
| `... is not a valid Win32 application`, `invalid ELF header`, `wrong architecture` | The binary does not match the platform of the JVM. Select it as shown above. |
| `Can't open a connection to same database file with a different configuration than existing connections` | Connections to one database file were opened with different properties. Pass `allow_unsigned_extensions` on all of them. |
| Load error mentioning the C API version | The JDBC driver is older than 1.5.6.0. Upgrade the driver. |
