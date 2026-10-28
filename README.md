![Version](https://img.shields.io/badge/version-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Depends](https://img.shields.io/badge/depends%20on-axiom-informational)
![License](https://img.shields.io/badge/license-Limited%20Commercial-red)

# Axiom Warp SQL Server Driver (`axiom-warp-sqlserver-driver`)

> The sovereign, framework-agnostic native ODBC (unixODBC) and Panama memory-mapped driver for Microsoft SQL Server.

`axiom-warp-sqlserver-driver` talks to SQL Server through the ODBC driver manager (`odbc32.dll` on Windows, `libodbc` / unixODBC elsewhere) plus Microsoft ODBC Driver for SQL Server using the Java Foreign Function & Memory API (Project Panama).
There is **no JDBC, no connection pool, no reflection and no third-party Java dependency** — just `axiom-spec`
(and transitively `axiom`) on the Java side and the vendor's native client on the other.

---

## Architecture overview

```
 ConfigSource ──► SqlServer.connect(config)  ─ curried provisioning ─►  SqlServer.Executioner
                                                                     │
          wield(conn -> …)  ── BoundScope(Arena.ofConfined) ──► engage() native connection
                                                                     │
           ┌──────────────────────────┬──────────────────────────────┼──────────────────────────┐
           ▼                          ▼                              ▼                          ▼
   shot(sql)                  dynamicQuery(sql)              fromFile(csv)               transaction(conn, …)
   Strike (raw SQL)           SqlParser + Strike             Hammer (bulk ingest)        commit / rollback
           │                  :java.name → native params     CsvEngine streaming
           └──────────────────────────┴──────────────┬───────────────┘
                                                     ▼
                         Result<PersistentList<PersistentMap<String, Object>>>
                                                     │
                                       disconnect()  ◄── always, success or failure
```

| Package (`com.ensemblu.axiom.warp.sqlserver.driver.*`) | Responsibility |
|---|---|
| `api.SqlServer` | Static `connect(..)` entry point and the `Executioner` operations gateway |
| `provision.RawProvisioner` | Validates configuration fields and materialises the `Executioner` |
| `engine.Strike` | Raw statements (`shot`) and parameterised queries (`dynamicQuery`) |
| `engine.SqlParser` | Rewrites `:java.name` markers into the native parameter syntax (max **21** per statement) |
| `ingest.Hammer` | Contract-verified bulk CSV ingestion |
| `io.CsvEngine` | Streaming CSV reader (classpath or filesystem) |
| `scope.BoundScope` | Binds a confined `Arena` to the current call via `ScopedValue`, freed when `wield` returns |
| `catalog.SqlServerType` | Native type catalog used for bindings and ingest contracts |

**Connection model.** Each `wield(..)` opens a dedicated native connection, runs your lambda, and closes the
connection and its arena before returning — nothing leaks between calls, and nothing is pooled. Keep work that must
share a session (transactions, temp tables) inside a single `wield`.

Dependency graph:

```
axiom  ◄──  axiom-spec  ◄──  axiom-warp-sqlserver-driver
```

---

## Installation

**Maven**

```xml
<dependency>
    <groupId>com.ensemblu</groupId>
    <artifactId>axiom-warp-sqlserver-driver</artifactId>
    <version>3.0.0</version>
</dependency>
```

**Gradle (Kotlin DSL)**

```kotlin
implementation("com.ensemblu:axiom-warp-sqlserver-driver:3.0.0")
```

JPMS automatic module name: `com.ensemblu.axiom.warp.sqlserver.driver`

### Runtime requirements

| Requirement | Value |
|---|---|
| JDK | **26**, run with `--enable-preview` |
| Native access | `--enable-native-access=ALL-UNNAMED` (avoids restricted-method warnings) |
| Native client | the ODBC driver manager (`odbc32.dll` on Windows, `libodbc` / unixODBC elsewhere) plus Microsoft ODBC Driver for SQL Server |

Looked up as `odbc32` on Windows (always present) and `odbc` elsewhere. Install **Microsoft ODBC Driver 18 for SQL Server** (or set `engine.odbc_driver` to the version you have).

---

## Configuration guide

Create a properties file on the classpath (e.g. `src/main/resources/database.properties`):

```properties
engine.host=localhost
engine.port=1433
engine.dbname=axiom_db
engine.user=axiom_commander
engine.password=strike_hard
engine.odbc_driver=ODBC Driver 18 for SQL Server
```

| Key | Required | Notes |
|---|---|---|
| `engine.host` | yes | Server host name or IP |
| `engine.port` | yes | TCP port (1433 by default) |
| `engine.dbname` | yes | Database |
| `engine.user` | yes | SQL login |
| `engine.password` | no | SQL login password |
| `engine.odbc_driver` | no | ODBC driver name; default `ODBC Driver 18 for SQL Server` |

Native connect string: `DRIVER={<odbc_driver>};SERVER=host,port;DATABASE=…;UID=…;PWD=…;TrustServerCertificate=yes;` via `SQLDriverConnect`.

Provisioning is a curried chain that mirrors the required fields, so a missing step is a compile error and a missing
or invalid value fails fast at boot with `🚨 INFRASTRUCTURE BOOT FAILURE`:

```java
import static com.ensemblu.axiom.api.Axiom.Config.file;
import com.ensemblu.axiom.warp.sqlserver.driver.api.SqlServer;

SqlServer.Executioner executioner = SqlServer
        .connect(file("database.properties"))//
        .requiredHost()//
        .requiredPort()//
        .requiredDbName()//
        .requiredUser()//
        .optionalPassword()//
        .getSqlServerDriverOrThrow();
```

Keep credentials out of source control: load from a file outside the repo, or build the config at runtime with
`Axiom.Config.source("engine.password=" + System.getenv("DB_PASSWORD") + "\n…")`.

---

## Usage examples

### Raw statements

```java
var rows = executioner.wield(conn ->
        executioner.shot("SELECT 1 AS ok").withConnection(conn));

rows.map(list -> list.first().toStringKeyMapVal().get("ok"));   // values arrive as text
```

T-SQL batches are allowed in a single `shot(..)`; `GO` separators are a client-tool feature and are not supported.

### Parameterised queries

```java
import com.ensemblu.axiom.warp.sqlserver.driver.catalog.SqlServerType;

Result<PersistentList<PersistentMap<String, Object>>> accounts =
        executioner.wield(conn ->
                executioner.dynamicQuery("""
                                SELECT TOP (:java.limit) account_id, balance
                                FROM accounts WHERE status = :java.status""")
                        .to(10).bind(SqlServerType.INT)
                        .to("ACTIVE").bind(SqlServerType.NVARCHAR)
                        .execute(conn));
```

Markers are rewritten to ODBC `?` parameters and bound out-of-band with `SQLBindParameter` using the C-type / SQL-type implied by the declared `SqlServerType`.

Rules: bind **every** marker, **in the order it appears** in the SQL; a repeated name is a separate marker. Result
values are returned as strings — convert at the edge (`row.targetKey("balance").toDoubleVal()`).

### Transactions

```java
Result<String> outcome = executioner.wield(conn ->
        executioner.transaction(conn, () -> {
            executioner.shot("UPDATE accounts SET status = 'FROZEN' WHERE owner = 'X'")
                       .withConnection(conn).getOrThrow();   // throwing → rollback
            executioner.shot("INSERT INTO audit_logs (action_type) VALUES ('FREEZE')")
                       .withConnection(conn).getOrThrow();
            return "committed";
        }));
```

Mechanism: `BEGIN TRANSACTION` / `COMMIT`, with rollback on any exception. The failure message is prefixed with `Transaction failed, rolled back:`.

For schema bootstrapping there is a shortcut that runs several statements in one transaction:

```java
executioner.wieldTransactional(
        () -> "CREATE TABLE accounts (...)",
        () -> "CREATE INDEX idx_accounts_status ON accounts(status)"
).getOrThrow();
```

### Bulk CSV ingestion

```java
import com.ensemblu.axiom.warp.sqlserver.driver.catalog.SqlServerType;

Result<Long> inserted = executioner.wield(conn ->
        executioner.fromFile("csv/initial_accounts.csv")        // classpath first, then filesystem
                   .withConnection(conn)
                   .usingFileHeaders()                          // or .withHeaders("owner", "balance", ...)
                   .withDataTypes(SqlServerType.NVARCHAR, SqlServerType.DECIMAL, SqlServerType.NVARCHAR, SqlServerType.NVARCHAR)
                   .onTableName("accounts"));
```

A prepared `INSERT` whose column buffers and length indicators are bound once with `SQLBindParameter`, then overwritten and re-executed per row, committed in batches of 1,000.

Declare exactly **one type per header, in header order**; header names must match column names exactly.

### Error handling

Every operation returns a `Result`; nothing throws unless you call `getOrThrow()`:

```java
accounts
    .peekFailure(e -> Axiom.Io.log("❌ " + e.getMessage()).run())
    .recover(e -> Axiom.Data.emptyList());
```

Pair with [`axiom-telemetry`](../../axiom-telemetry) to attach a pre-failure snapshot to every database failure.

---

## License

**Limited Commercial License** — free for evaluation, testing, and non-commercial development.
Commercial or production use requires a paid annual contract.
See [LICENSE](https://raw.githubusercontent.com/ensemblu-corp/axiom/main/LICENSE) · Contact: contact@ensemblu.com

© Ensemblu Corp
