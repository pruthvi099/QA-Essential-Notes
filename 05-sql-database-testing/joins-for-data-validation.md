# Joins for Data Validation

## What It Is

A `JOIN` combines rows from two or more tables based on a related column, letting a single query verify relationships across tables — e.g., confirming every order line item points to a real, existing product. This extends [SQL Fundamentals for QA](./sql-fundamentals-for-qa.md) into the multi-table validation scenarios that come up constantly once an application has more than one related table (which is nearly always).

## Why It Matters

- Real applications almost never store data in a single flat table — orders relate to customers, items relate to products, permissions relate to roles. Validating data integrity across these relationships requires joins; single-table queries structurally can't check them.
- Joins are how you catch **referential integrity** bugs — an order line item referencing a deleted product, an order with no matching customer record — which are exactly the kind of subtle, high-impact data bugs that don't show up in a UI test but corrupt real business data.
- Alongside basic `SELECT`/`WHERE`, join fluency is one of the most consistently tested SQL skills in SDET interviews, since it's directly practical for real backend validation work.

## How It Works

**Join types relevant to QA validation:**

| Join Type | Returns |
|---|---|
| `INNER JOIN` | Only rows with a match in both tables |
| `LEFT JOIN` | All rows from the left table, plus matches from the right (NULL where no match) |
| `RIGHT JOIN` | All rows from the right table, plus matches from the left (NULL where no match) |
| `FULL OUTER JOIN` | All rows from both tables, matched where possible, NULL where not |

**The QA-specific pattern:** `LEFT JOIN` is the most useful join for data integrity checks specifically — it lets you find rows in the "left" table that have **no matching row** in the related table, by filtering for `NULL` in the joined columns. This is how you detect orphaned records.

## Example

```sql
-- INNER JOIN: get order + customer details together (basic combined lookup)
SELECT o.id AS order_id, o.total, c.email, c.tier
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.id = 501;

-- LEFT JOIN data integrity check: find orders with NO matching customer —
-- this should be IMPOSSIBLE if referential integrity is correctly enforced,
-- so any result here is a real, serious data bug
SELECT o.id, o.customer_id
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;

-- LEFT JOIN: find order line items referencing a product that no longer exists
SELECT oi.id AS line_item_id, oi.order_id, oi.product_id
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.id
WHERE p.id IS NULL;

-- Multi-table join: verify order total matches the sum of its line items
-- (a cross-table business logic check, not just structural integrity)
SELECT o.id, o.total AS stored_total, SUM(oi.price * oi.qty) AS calculated_total
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.total
HAVING o.total != SUM(oi.price * oi.qty);
```

Using the orphaned-record check in an automated test:

```python
def test_no_orders_reference_missing_customers(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("""
        SELECT o.id, o.customer_id
        FROM orders o
        LEFT JOIN customers c ON o.customer_id = c.id
        WHERE c.id IS NULL
    """)
    orphaned_orders = cursor.fetchall()

    assert len(orphaned_orders) == 0, (
        f"Found {len(orphaned_orders)} orders referencing non-existent customers: "
        f"{orphaned_orders}"
    )
```

## Production Considerations

- The "LEFT JOIN + WHERE right-side IS NULL" pattern is worth having ready as a standard tool — it's the most direct way to find orphaned/broken foreign-key relationships, and comes up repeatedly across different table pairs in any real schema.
- Running orphaned-record checks periodically (not just once) against a staging/test environment can catch data integrity regressions introduced by application bugs before they're discovered through a customer-facing symptom.
- Be mindful of join performance on large tables in non-test environments — joins without proper indexes on the joined columns can be slow; this matters less in typical test/staging data volumes but is worth knowing if validation queries are ever run against production-scale data.

## Common Pitfalls

- Using `INNER JOIN` when checking for orphaned/missing records — `INNER JOIN` only returns matching rows, so it structurally can't reveal what's *missing*; `LEFT JOIN` (with a `WHERE ... IS NULL` filter) is required for that specific check.
- Forgetting the `WHERE right_table.id IS NULL` filter after a `LEFT JOIN`, resulting in a query that returns all rows (matched and unmatched) instead of isolating just the broken/orphaned ones.
- Not accounting for `GROUP BY` requirements when combining joins with aggregations (as in the order-total example) — omitting a selected non-aggregated column from `GROUP BY` causes a SQL error or unpredictable results depending on the database engine.
- Assuming referential integrity is enforced by the database (foreign key constraints) without verifying it — some schemas don't enforce foreign keys at the DB level, relying entirely on application logic, which makes these validation queries even more important as a safety net.

## Interview Notes

- Be ready to write a `LEFT JOIN` query to find orphaned records — a very common, practical SQL interview exercise that directly tests QA-relevant join usage, not just join syntax in the abstract.
- Understand precisely why `LEFT JOIN` (not `INNER JOIN`) is the correct tool for finding missing/orphaned relationships — this is often a specific follow-up question distinguishing genuine understanding from memorized syntax.
- Be able to explain a real scenario where a join-based validation query caught a data integrity bug a single-table check couldn't have found.

## References

- [PostgreSQL Documentation — Joins](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
- [MySQL Documentation — JOIN Syntax](https://dev.mysql.com/doc/refman/8.0/en/join.html)