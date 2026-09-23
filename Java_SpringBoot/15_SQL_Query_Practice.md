# SQL Query Practice

Fills the empty SQL section in `Technical.pdf`. `09_Database_Transactions.md` covers ACID/isolation/locking theory — this file is about **writing queries**, which is what an actual SQL round tests. Most Java backend loops include one, and it's usually a shared editor with a couple of table definitions and 3–5 questions of increasing difficulty.

Assume these tables throughout:

```sql
employees(id, name, salary, dept_id, manager_id, hired_on)
departments(id, name)
orders(id, customer_id, amount, created_at, status)
```

## Joins — the thing most people get slightly wrong

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows with a match on both sides |
| `LEFT JOIN` | All left rows; NULLs where no right match |
| `RIGHT JOIN` | All right rows; NULLs where no left match |
| `FULL OUTER JOIN` | All rows from both, NULLs where unmatched |
| `CROSS JOIN` | Cartesian product |
| Self join | A table joined to itself (aliased twice) |

Two traps worth knowing cold:

**1. A `WHERE` clause on the right-hand table silently turns a LEFT JOIN into an INNER JOIN.**

```sql
-- BROKEN: rows with no order have o.status = NULL, and NULL != 'SHIPPED' filters them out
SELECT c.name, o.id FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'SHIPPED';

-- CORRECT: put the condition in the JOIN so unmatched rows survive
SELECT c.name, o.id FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'SHIPPED';
```

**2. Self join for hierarchy** — the standard "list each employee with their manager's name":

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;   -- LEFT, so the CEO (no manager) still appears
```

## GROUP BY, HAVING, and the aggregation gotchas

- `WHERE` filters **rows before** grouping; `HAVING` filters **groups after** aggregation. You cannot use an aggregate in `WHERE`.
- Logical execution order (not the written order — this explains most confusion): `FROM` → `JOIN` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`. Because `SELECT` runs after `GROUP BY`, a column alias defined in `SELECT` can be used in `ORDER BY` but generally not in `WHERE`.
- **`COUNT(*)` counts rows including NULLs; `COUNT(column)` skips NULLs.** A frequent trick question.
- Aggregates other than `COUNT(*)` **ignore NULLs**, so `AVG(salary)` divides by the count of non-null salaries, not the row count.

```sql
-- Departments with more than 5 employees, by average salary
SELECT d.name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON d.id = e.dept_id
GROUP BY d.name
HAVING COUNT(*) > 5
ORDER BY avg_salary DESC;
```

## Window functions — the biggest differentiator

If you learn one thing from this file, learn these. Candidates who reach for window functions instead of correlated subqueries stand out immediately.

A window function computes across a set of rows **without collapsing them** — unlike `GROUP BY`, every input row remains in the output.

```sql
function() OVER (PARTITION BY col ORDER BY col ROWS BETWEEN ... )
```

**Ranking — know the difference between these three, it's asked directly:**

| Function | Ties get | Gaps after ties? |
|---|---|---|
| `ROW_NUMBER()` | Distinct arbitrary numbers | N/A |
| `RANK()` | Same rank | **Yes** (1,1,3) |
| `DENSE_RANK()` | Same rank | **No** (1,1,2) |

```sql
SELECT name, dept_id, salary,
       ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
       RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense
FROM employees;
```

**`LAG` / `LEAD`** — reach into the previous/next row, for month-over-month deltas and gap detection:

```sql
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       revenue - LAG(revenue) OVER (ORDER BY month) AS delta
FROM monthly_revenue;
```

**Running totals** — a frame clause over an ordered partition:

```sql
SELECT created_at, amount,
       SUM(amount) OVER (ORDER BY created_at
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

## The standard interview questions

### Nth highest salary

The canonical question. Give the window-function answer, then mention the alternatives.

```sql
-- 2nd highest per department, handling ties correctly
SELECT * FROM (
  SELECT name, dept_id, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk = 2;
```

Ask the interviewer whether ties should count as one rank (`DENSE_RANK`) or consume ranks (`RANK`) — noticing the ambiguity is part of what's being scored. The naive `LIMIT 1 OFFSET 1` approach breaks on ties and can't do it per-group.

### Find duplicates

```sql
SELECT email, COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

Follow-up — **delete duplicates keeping the earliest row**:

```sql
DELETE FROM users
WHERE id NOT IN (SELECT MIN(id) FROM users GROUP BY email);
```

(Some engines dislike reading and deleting the same table in a subquery; the portable alternative uses `ROW_NUMBER()` in a CTE and deletes where `rn > 1`.)

### Employees earning more than their manager

```sql
SELECT e.name
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### Departments with no employees

```sql
SELECT d.name FROM departments d
LEFT JOIN employees e ON e.dept_id = d.id
WHERE e.id IS NULL;
```

**Never write `NOT IN` against a nullable subquery** — if the subquery returns a single NULL, `NOT IN` yields no rows at all (NULL comparisons are unknown, never true). `NOT EXISTS` or the `LEFT JOIN ... IS NULL` form above are safe. This is a genuinely good trap and worth raising unprompted.

### Top N per group

Same `ROW_NUMBER()` shape as Nth-highest with `WHERE rn <= N`. Recognizing that these are the same problem is the point.

### Consecutive/gap problems

"Find users who logged in 3 days in a row" — solve with `LAG`, or the *gaps-and-islands* trick: subtract `ROW_NUMBER()` from the date, and consecutive dates produce a constant value you can group on.

## Subqueries and CTEs

- **Scalar subquery** — returns one value, usable in `SELECT`/`WHERE`.
- **Correlated subquery** — references the outer query, re-evaluated per outer row. Readable but often slow; usually rewritable as a join or window function, which is the optimization interviewers look for.
- **CTE (`WITH`)** — names a subquery, improving readability and allowing reuse. **Recursive CTEs** (`WITH RECURSIVE`) walk hierarchies — the natural answer to "print the full org chart under this manager".

```sql
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE id = 1                    -- anchor
    UNION ALL
    SELECT e.id, e.name, e.manager_id, org.level + 1
    FROM employees e JOIN org ON e.manager_id = org.id   -- recursive step
)
SELECT * FROM org;
```

## Indexing & EXPLAIN (crossover with the transactions file)

- A B-tree index speeds up equality/range lookups, joins, and `ORDER BY` at the cost of slower writes and extra storage.
- **Leftmost prefix rule:** an index on `(a, b, c)` serves filters on `a`, `a,b`, or `a,b,c` — but not `b` alone.
- **A function on an indexed column disables the index:** `WHERE YEAR(created_at) = 2026` can't use an index on `created_at`. Rewrite as a range: `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'`. Same for leading-wildcard `LIKE '%foo'`. This is the single most common real-world "why is my query slow" answer.
- **Covering index** — contains every column the query needs, so it's satisfied from the index alone (index-only scan).
- `EXPLAIN` / `EXPLAIN ANALYZE` is how you verify rather than guess. Say you'd check it — "I'd look at the plan to confirm it's an index scan and not a seq scan" is a strong sentence in a performance question.

## Quick reference — things that trip people up

- `NULL` is never equal to anything, including `NULL`. Use `IS NULL` / `IS NOT NULL`. `COALESCE(x, fallback)` substitutes a default.
- `UNION` removes duplicates (and sorts to do so); `UNION ALL` doesn't — prefer `UNION ALL` when you know there are no duplicates, it's meaningfully faster.
- `DELETE` is row-by-row, logged, transactional, fires triggers, and can be rolled back. `TRUNCATE` deallocates whole pages, is much faster, resets identity/sequence, and generally can't be rolled back. `DROP` removes the table itself.
- `CHAR` is fixed-width and space-padded; `VARCHAR` is variable-length.
- `WHERE` before grouping, `HAVING` after. `LIMIT`/`OFFSET` last — and deep `OFFSET` pagination gets slow, which is why keyset ("seek") pagination on an indexed column is preferred at scale.

## Commonly asked

- Write a query for the 2nd (or Nth) highest salary per department. What happens with ties?
- `RANK()` vs `DENSE_RANK()` vs `ROW_NUMBER()` — when does the difference matter?
- `WHERE` vs `HAVING` — why can't you use an aggregate in `WHERE`?
- Why does adding a `WHERE` on the right table break a `LEFT JOIN`?
- Why is `NOT IN` dangerous with a nullable subquery?
- Find and then delete duplicate rows, keeping one.
- Why doesn't this query use my index? (Function on the column / leading wildcard / wrong composite column order.)
- `DELETE` vs `TRUNCATE` vs `DROP`.
- Compute a running total and a month-over-month change.
