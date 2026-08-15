# SQL Fundamentals for QA

## What It Is

This note covers the subset of SQL an SDET actually needs day to day: querying data to verify application behavior, not building or optimizing a database. The focus is `SELECT`, `WHERE`, sorting, and basic filtering — the foundation everything else in this folder (joins, aggregations, seeding) builds on.

## Why It Matters

- Most application bugs eventually need a database-level check to confirm root cause — "is the data actually wrong, or is it a display/API bug?" — and answering that requires being comfortable writing ad-hoc SQL queries quickly, not looking up syntax mid-investigation.
- SQL fluency is one of the most commonly and directly tested skills in SDET interviews — candidates are frequently given a schema and asked to write a query live, making this genuinely practical, not just theoretical.
- This underpins [Database Validation from API Tests](../04-api-testing/database-validation-from-api-tests.md) and every later note in this folder — without comfortable `SELECT`/`WHERE` fluency, none of the more advanced validation patterns are practical to write quickly.

## How It Works

**Core query structure:**
```sql
SELECT column1, column2
FROM table_name
WHERE condition
ORDER BY column1 [ASC|DESC]
LIMIT n;
```

**Common `WHERE` conditions for QA validation:**
- `=`, `!=`, `>`, `<`, `>=`, `<=` — standard comparisons
- `IN (...)` — matching against a set of values
- `LIKE '%pattern%'` — pattern matching (useful for checking data format issues)
- `IS NULL` / `IS NOT NULL` — checking for missing data, a very common QA validation need
- `BETWEEN x AND y` — range checks (useful for boundary-value validation, see [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md))
- `AND` / `OR` — combining conditions

## Example

Common QA validation queries against an orders table:

```sql
-- Find a specific order by ID (basic lookup, most common query type)
SELECT id, customer_email, status, total
FROM orders
WHERE id = 501;

-- Find all orders in an unexpected/invalid state — a data integrity check
SELECT id, status
FROM orders
WHERE status NOT IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled');

-- Find orders with a NULL total — should never happen if the app is correct
SELECT id, customer_email
FROM orders
WHERE total IS NULL;

-- Find orders created in a specific test run's time window
SELECT id, customer_email, created_at
FROM orders
WHERE created_at BETWEEN '2026-08-14 00:00:00' AND '2026-08-14 23:59:59'
ORDER BY created_at DESC;

-- Find test accounts by email pattern, useful for verifying test data cleanup
SELECT email, created_at
FROM users
WHERE email LIKE 'test_%@example.com'
ORDER BY created_at DESC
LIMIT 20;

-- Verify no order has a negative total — a boundary/invariant check
SELECT id, total
FROM orders
WHERE total < 0;
```

Using this from a test, via Playwright's `APIRequestContext` pattern combined with a DB connection (see [Database Validation from API Tests](../04-api-testing/database-validation-from-api-tests.md) for the full fixture setup):

```python
def test_no_orders_have_negative_totals(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT id, total FROM orders WHERE total < 0")
    invalid_orders = cursor.fetchall()

    assert len(invalid_orders) == 0, f"Found orders with negative totals: {invalid_orders}"
```

## Production Considerations

- Always use a read-only database user/role for QA validation queries — this is a security and safety boundary, not just a best practice, and prevents an accidental typo in a query from modifying real data (see [Database Validation from API Tests](../04-api-testing/database-validation-from-api-tests.md)).
- `LIKE '%pattern%'` queries (leading wildcard) can be slow on large tables since they typically can't use an index efficiently — fine for test/staging environments with modest data volumes, but worth being aware of if querying against a large, production-scale dataset.
- Always include a `LIMIT` when exploring/debugging interactively against a large table — an unbounded `SELECT *` against a production-scale table can be slow and resource-intensive even in a read-only context.

## Common Pitfalls

- Using `SELECT *` in automated test assertions instead of naming specific columns — this makes tests fragile to schema changes (a new column breaks nothing functionally but can complicate result-shape assumptions) and less clear about what's actually being validated.
- Forgetting `IS NULL`/`IS NOT NULL` are required for NULL comparisons — `WHERE total = NULL` never matches anything in standard SQL, a common, confusing gotcha for those newer to SQL.
- Writing overly broad `WHERE` clauses in validation queries that unintentionally match legitimate data alongside the actual issue being investigated, producing a confusing or misleading result set.
- Not using `ORDER BY` when investigating recent test data, making it hard to distinguish current test run results from stale leftover data.

## Interview Notes

- Be ready to write a `SELECT` query with `WHERE` filtering live, given a described schema — an extremely common opening SQL interview exercise.
- Understand why `= NULL` doesn't work and `IS NULL` is required — a small but frequently tested SQL correctness detail.
- Be able to explain a real scenario where you used SQL to diagnose whether a bug was in the application layer or the data layer — connects this fundamental skill to practical debugging value.

## References

- [PostgreSQL Documentation — SELECT](https://www.postgresql.org/docs/current/sql-select.html)
- [MySQL Documentation — SELECT Statement](https://dev.mysql.com/doc/refman/8.0/en/select.html)