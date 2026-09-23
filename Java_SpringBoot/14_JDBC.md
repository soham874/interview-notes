# JDBC

Worth knowing even though you'll rarely write raw JDBC in a Spring app — interviewers use it to check you understand what JPA/Hibernate is doing underneath, and connection pooling questions come up constantly in performance discussions.

## What JDBC is

A standard Java API (`java.sql`) for talking to relational databases. Its value is **vendor independence**: your code targets the JDBC interfaces, and a vendor-supplied **driver** implements them for a specific database. Swap Postgres for MySQL and, in principle, only the driver jar and connection URL change.

## API components

**Interfaces** (implemented by the driver):

- `Driver` — the vendor's entry point.
- `Connection` — a session with the database. Statements execute within its context; also carries transaction control (`setAutoCommit`, `commit`, `rollback`) and metadata (`getMetaData()` → tables, supported SQL grammar, stored procedures).
- `Statement` — executes static SQL.
- `PreparedStatement` — precompiled, parameterized SQL (extends `Statement`).
- `CallableStatement` — invokes stored procedures (extends `PreparedStatement`).
- `ResultSet` — cursor over query results.

**Classes:**

- `DriverManager` — the legacy factory: tracks registered drivers and hands out connections for a URL. **`DataSource` is the modern replacement** and what Spring uses — it supports pooling, distributed transactions, and container-managed configuration, none of which `DriverManager` does.
- `SQLException` — checked exception for database errors.

## Class.forName() — and why you no longer need it

`Class.forName("com.mysql.cj.jdbc.Driver")` explicitly loads a driver class, whose static initializer registers it with `DriverManager`.

**Since JDBC 4.0 (Java 6) this is unnecessary** — drivers are auto-discovered via the service-provider mechanism (`META-INF/services/java.sql.Driver` on the classpath). Seeing `Class.forName` in modern code is a sign of a tutorial written in 2005. Good thing to say if it comes up: know what it did, know it's obsolete.

## Statement vs PreparedStatement vs CallableStatement

| | `Statement` | `PreparedStatement` |
|---|---|---|
| SQL | Built by string concatenation | Parameterized with `?` placeholders |
| Compilation | Parsed/planned on every execution | Precompiled once, reused with new parameters |
| Performance | Slower when repeated | Faster on repeated execution |
| **SQL injection** | **Vulnerable** | **Safe** — parameters are bound, never parsed as SQL |
| Binary/large data | Awkward | `setBlob`, `setBytes`, etc. |

**Always prefer `PreparedStatement`.** The performance argument is real but secondary — the headline reason is SQL injection safety, and that's the answer interviewers want first.

```java
// Vulnerable — never do this
String sql = "SELECT * FROM users WHERE email = '" + email + "'";

// Safe — the driver binds the value; it can never be interpreted as SQL
String sql = "SELECT * FROM users WHERE email = ?";
try (PreparedStatement ps = connection.prepareStatement(sql)) {
    ps.setString(1, email);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { /* ... */ }
    }
}
```

Note the parameter index is **1-based**, and `try-with-resources` closes `Connection`/`Statement`/`ResultSet` in reverse order automatically — forgetting to close them is a leading cause of connection-pool exhaustion.

**`CallableStatement`** executes stored procedures: `connection.prepareCall("{call get_user_orders(?, ?)}")`, with `registerOutParameter()` for OUT params. Stored procedures push logic into the database — faster for set-heavy work and centrally controlled, but harder to version-control, test, and migrate between vendors. Most teams keep logic in the application these days.

## Batch processing

Instead of a network round-trip per statement, group them and send once:

```java
try (PreparedStatement ps = conn.prepareStatement("INSERT INTO events(name, ts) VALUES (?, ?)")) {
    for (Event e : events) {
        ps.setString(1, e.getName());
        ps.setTimestamp(2, e.getTimestamp());
        ps.addBatch();
        if (++count % 1000 == 0) ps.executeBatch();   // flush periodically
    }
    ps.executeBatch();   // remaining
}
```

The win is eliminating per-statement round-trips — often an order of magnitude on bulk inserts. Flush in chunks rather than accumulating everything, or you trade network time for memory. In JPA the equivalent knobs are `hibernate.jdbc.batch_size` plus `order_inserts`/`order_updates`, and you must also `flush()`/`clear()` the persistence context periodically or the first-level cache grows unboundedly.

## Connection pooling (the part that actually matters in interviews)

Opening a database connection is **expensive** — TCP handshake, authentication, session setup, often 10–100ms. Doing that per request is untenable under load.

A **connection pool** opens a set of connections at startup and keeps them open. "Getting a connection" becomes borrowing an idle one from the pool; "closing" it returns it to the pool rather than tearing it down.

- **HikariCP** is the default in Spring Boot and the standard answer. (Older: Tomcat JDBC pool, C3P0, DBCP.)
- Key settings: `maximum-pool-size`, `minimum-idle`, `connection-timeout` (how long a caller waits for a free connection before failing), `idle-timeout`, `max-lifetime` (recycle connections before the DB or a firewall kills them).
- **Sizing is counterintuitive.** Bigger is not better — a common guideline is roughly `(core_count × 2) + effective_spindle_count`, so pools in the low tens, not hundreds. Too many connections cause context-switching and lock contention *inside the database*, reducing throughput. The pool also must not exceed the database's own `max_connections`, especially with many app instances.
- **Pool exhaustion** is the classic production incident: every connection is checked out, new requests block on `connection-timeout` and then fail. Usual causes — connections not closed (missing `try-with-resources`), transactions held open across slow external calls, or a long-running query monopolizing the pool. This is a very common "tell me about a production issue" prompt.

## How this maps to Spring

You almost never touch `DriverManager`. Spring Boot auto-configures a `DataSource` (HikariCP) from `spring.datasource.*` properties, and you work at one of three levels:

- **`JdbcTemplate`** — Spring's Template-pattern wrapper over raw JDBC. Handles connection acquisition/release, statement creation, `ResultSet` iteration, and translates `SQLException` into Spring's `DataAccessException` hierarchy (unchecked, vendor-neutral). You supply the SQL and a `RowMapper`.
- **`JdbcClient`** (Spring 6.1+) — a fluent modern API over the same machinery; nicer than `JdbcTemplate` for new code.
- **Spring Data JPA / Hibernate** — full ORM on top; see `08_Spring_Data_JPA_Hibernate.md`.

```java
List<User> users = jdbcTemplate.query(
    "SELECT id, name FROM users WHERE status = ?",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),
    status);
```

Good interview framing for "JPA or raw SQL?": JPA for straightforward entity CRUD where the mapping earns its keep; `JdbcTemplate`/`JdbcClient` for reporting queries, complex joins, and bulk operations where you want exact control over the SQL and don't want an object graph. Mixing both in one codebase is normal and not a design smell.

## Commonly asked

- Why is `PreparedStatement` preferred over `Statement`? Give both reasons, in priority order.
- What is connection pooling, and why is a pool of 200 connections usually worse than a pool of 20?
- You're paged because an app is timing out and the DB looks idle. How do you diagnose connection pool exhaustion?
- What does `Class.forName()` do in JDBC, and why doesn't modern code need it?
- How does batch processing improve performance, and what's the tradeoff of a huge batch size?
- What does `JdbcTemplate` do for you that raw JDBC doesn't?
- When would you drop down from JPA to `JdbcTemplate`?
