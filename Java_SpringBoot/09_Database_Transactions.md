# Database & Transactions

## ACID

- **Atomicity** — a transaction is all-or-nothing; a failure partway rolls back everything.
- **Consistency** — a transaction moves the DB from one valid state to another, respecting constraints (FKs, unique, check constraints).
- **Isolation** — concurrent transactions don't see each other's uncommitted intermediate state (degree depends on isolation level, below).
- **Durability** — once committed, a transaction's changes survive a crash (typically via write-ahead logging).

## Isolation levels & the anomalies they prevent

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible (mostly — some DBs like Postgres actually prevent phantoms too here in practice) |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read**: reading another transaction's uncommitted changes (which might get rolled back).
- **Non-repeatable read**: re-reading the same row within a transaction gives a different value because another transaction committed a change in between.
- **Phantom read**: re-running the same range query within a transaction returns a different *set of rows* because another transaction inserted/deleted matching rows in between.
- Most relational DBs default to **Read Committed** (Postgres, Oracle, SQL Server) or **Repeatable Read** (MySQL/InnoDB). Higher isolation = more correctness guarantees but more locking/contention and lower throughput — this tradeoff is the actual point interviewers want you to articulate, not just the table.

## Transaction propagation (Spring's `@Transactional(propagation = ...)`)

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join the current transaction if one exists, else create a new one |
| `REQUIRES_NEW` | Always start a new, independent transaction — suspends the current one if present |
| `NESTED` | Runs within a savepoint of the current transaction — can roll back just the nested part without failing the outer transaction (DB-dependent support) |
| `SUPPORTS` | Join if a transaction exists, else run non-transactionally |
| `MANDATORY` | Must run within an existing transaction, throws if none |
| `NEVER` | Must run without a transaction, throws if one exists |

Practical example: an audit-log write that must persist even if the surrounding business transaction rolls back → `REQUIRES_NEW` for that specific method, so its own commit is independent.

## Locking

- **Optimistic locking** — no lock held; instead, detect conflict at write time via a version column (or timestamp) — if the row changed since you read it, the update fails/conflicts. Good for low-contention, high-read workloads; requires a retry strategy on conflict.
- **Pessimistic locking** — acquire a lock at read time (`SELECT ... FOR UPDATE`) so no one else can modify the row until you're done. Reduces conflicts but increases contention/blocking, and can cause deadlocks if lock acquisition order isn't consistent across transactions.
- Row-level vs table-level locks — most modern relational engines default to row-level locking for DML, which is why index design matters (a query without a usable index can escalate to scanning/locking far more rows than intended).

## Indexing basics (enough to hold a design conversation)

- A B-tree index (the default for most range/equality lookups) speeds up `WHERE`, `JOIN`, `ORDER BY` on the indexed column(s) at the cost of slower writes (every insert/update/delete has to maintain the index too) and extra storage.
- Composite indexes are ordered — an index on `(a, b)` helps queries filtering on `a` alone or `a AND b`, but not on `b` alone (leftmost prefix rule).
- Covering index — an index that includes all columns a query needs, letting the DB satisfy the query from the index alone without touching the table (index-only scan).
- `EXPLAIN`/`EXPLAIN ANALYZE` is the tool to actually verify an index is being used rather than guessing — worth mentioning if asked how you'd diagnose a slow query.

## Common gotchas to be ready for

- Why `@Transactional` with default settings silently doesn't roll back on a checked exception (see JPA notes) — a favorite "gotcha" question.
- Why a long-running transaction is dangerous even if logically correct — holds locks longer, increases contention, can hold back MVCC garbage collection (e.g., Postgres autovacuum bloat) on some databases.
- Batch operations and transaction scope — doing 10,000 inserts in one giant transaction vs. batching into smaller committed chunks — tradeoff between atomicity and lock duration/memory.

## Commonly asked

- Explain the four ACID properties with a concrete example of what breaks if one is missing.
- What's the difference between a dirty read, non-repeatable read, and phantom read?
- What isolation level does [Postgres/MySQL] default to, and what tradeoff does raising it introduce?
- When would you use `REQUIRES_NEW` instead of the default `REQUIRED` propagation?
- Optimistic vs pessimistic locking — walk through a scenario where each is the right call.
- What's the leftmost prefix rule for composite indexes, and why does column order matter?
