# Aggregations & Data Integrity Checks

## What It Is

Aggregation functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) combined with `GROUP BY` let you validate data at a summary level — detecting duplicates, verifying totals across many rows at once, and catching integrity violations that a row-by-row check would be far too slow to find manually. This extends [Joins for Data Validation](./joins-for-data-validation.md) into checks that operate across many rows simultaneously rather than validating individual relationships one at a time.

## Why It Matters

- Some of the most damaging data bugs are aggregate-level, not visible from any single row — a duplicate order silently double-counted in revenue reporting, an inventory count that's drifted from the sum of actual transactions — these only surface through aggregation queries.
- `GROUP BY ... HAVING COUNT(*) > 1` is one of the single most useful QA queries that exists — it's the standard way to find duplicate records that should be unique (duplicate emails, duplicate order IDs), a very common real-world data integrity check.
- These patterns are frequently asked about directly in SDET interviews ("write a query to find duplicate emails") because they're both common in practice and demonstrate genuine SQL fluency beyond basic `SELECT`/`WHERE`.

## How It Works

**Core aggregation functions:**
- `COUNT(*)` — count of rows (or `COUNT(column)` — count of non-NULL values in that column)
- `SUM(column)` — total of a numeric column
- `AVG(column)` — average of a numeric column
- `MIN(column)` / `MAX(column)` — smallest/largest value

**`GROUP BY`** groups rows sharing a value in specified column(s), applying the aggregate function per group rather than across the whole table.

**`HAVING`** filters *after* aggregation (unlike `WHERE`, which filters *before* grouping) — this is the key to finding groups that violate an expected rule, like "should never have more than 1 row."

## Example

```sql
-- Find duplicate customer emails — should be unique, so any group with
-- COUNT > 1 represents a real data integrity violation
SELECT email, COUNT(*) AS occurrence_count
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;

-- Verify total revenue matches expectations for a given day —
-- a sanity check comparing aggregate data against a known expected value
SELECT DATE(created_at) AS order_date, SUM(total) AS daily_revenue, COUNT(*) AS order_count
FROM orders
WHERE status = 'paid'
GROUP BY DATE(created_at)
ORDER BY order_date DESC
LIMIT 7;

-- Find products with an average order quantity that seems suspiciously high
-- (could indicate a bulk-order abuse pattern, or a units bug in the app)
SELECT product_id, AVG(qty) AS avg_qty, MAX(qty) AS max_qty
FROM order_items
GROUP BY product_id
HAVING AVG(qty) > 50
ORDER BY avg_qty DESC;

-- Verify every customer has at least one order (a business assumption check,
-- distinct from a referential integrity check — this validates a business rule)
SELECT c.id, c.email, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.email
HAVING COUNT(o.id) = 0;
```

Using the duplicate-detection query as an automated data integrity test:

```python
def test_no_duplicate_customer_emails(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("""
        SELECT email, COUNT(*) AS occurrence_count
        FROM customers
        GROUP BY email
        HAVING COUNT(*) > 1
    """)
    duplicates = cursor.fetchall()

    assert len(duplicates) == 0, f"Found duplicate customer emails: {duplicates}"

def test_daily_revenue_is_reasonable(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("""
        SELECT SUM(total) AS daily_revenue
        FROM orders
        WHERE status = 'paid' AND DATE(created_at) = CURRENT_DATE
    """)
    revenue = cursor.fetchone()[0] or 0

    # A sanity bound, not an exact expected value — catches gross errors
    # (e.g., a units bug causing 100x inflated totals) without being
    # brittle to normal day-to-day revenue fluctuation
    assert 0 <= revenue < 10_000_000, f"Daily revenue ({revenue}) outside sane bounds"
```

## Production Considerations

- Aggregate-level checks like duplicate detection and revenue sanity bounds are excellent candidates for scheduled monitoring queries (run periodically against production, not just test environments) — they catch data drift and integrity issues that emerge gradually over time, not just at the moment of a specific test.
- Be careful with `COUNT(column)` vs. `COUNT(*)` — the former excludes NULLs in that specific column, the latter counts all rows regardless of NULLs; using the wrong one silently produces a miscounted result that's easy to miss.
- When validating totals/revenue with a sanity-bound test (as shown above) rather than an exact value, choose bounds wide enough to tolerate normal variation but tight enough to actually catch a real bug — this requires some domain knowledge of what "normal" looks like for the specific business.

## Common Pitfalls

- Using `WHERE` instead of `HAVING` to filter on an aggregated value — `WHERE` executes before aggregation and can't reference `COUNT(*)`/`SUM(...)` directly, a very common SQL syntax error for those newer to aggregation queries.
- Forgetting to include all non-aggregated selected columns in `GROUP BY` — most SQL engines either error or produce unpredictable results when a selected column is neither aggregated nor included in the grouping.
- Confusing `COUNT(*)` (all rows) with `COUNT(specific_column)` (non-NULL values only) — using the wrong one when checking for missing data specifically produces a subtly wrong result.
- Writing an aggregate sanity check with bounds so wide they'd never actually catch a real regression, defeating the purpose of the check while still looking like meaningful test coverage.

## Interview Notes

- Be ready to write a `GROUP BY ... HAVING COUNT(*) > 1` query to find duplicates — one of the most commonly asked practical SQL exercises in SDET interviews.
- Understand the `WHERE` vs. `HAVING` distinction precisely (filters before vs. after aggregation) — a frequently tested, easy-to-get-wrong SQL fundamental.
- Be able to describe a real (or plausible) scenario where an aggregate-level check caught a data integrity bug that row-by-row testing wouldn't have surfaced.

## References

- [PostgreSQL Documentation — Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [MySQL Documentation — GROUP BY](https://dev.mysql.com/doc/refman/8.0/en/group-by-handling.html)