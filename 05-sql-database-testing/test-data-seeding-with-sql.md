# Test Data Seeding with SQL

## What It Is

Test data seeding via SQL means writing scripts that populate a database with a known, consistent baseline state before a test cycle runs — directly, rather than through the application's API or UI. This complements [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) and [API Test Data Management](../04-api-testing/api-test-data-management.md) by covering the SQL-specific tooling: `INSERT` scripts, bulk seed files, and reset scripts that establish a repeatable starting point.

## Why It Matters

- API-based seeding (covered in [API Test Data Management](../04-api-testing/api-test-data-management.md)) is realistic but slow for bulk setup — seeding hundreds of baseline records through individual API calls is impractically slow compared to a single bulk `INSERT` script, especially for large regression environments that need to be reset frequently.
- A reliable, repeatable seed script is the foundation of a stable test environment — without it, tests build on whatever leftover state happens to exist, causing exactly the kind of flaky, hard-to-reproduce failures covered in [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md).
- Being able to write efficient seed/reset scripts (not just individual test-data-creation calls) is a practical, expected SDET skill for setting up shared staging or CI database environments from scratch.

## How It Works

**Core approaches:**
1. **Bulk `INSERT` scripts** — directly populate baseline reference data (product catalog, default roles, standard test accounts) that rarely changes between test runs.
2. **Reset scripts** — `TRUNCATE`/`DELETE` + re-seed, run before a test cycle to guarantee a known, consistent starting state, undoing any drift from previous test runs.
3. **Fixtures/factories at the SQL level** — parameterized `INSERT` templates for generating realistic volumes of related data (e.g., 50 test orders across 10 test customers) without hand-writing each row.

**When SQL seeding is the right tool vs. API-based seeding:** use SQL directly for bulk, foundational baseline data (reference/lookup tables, a large volume of historical-looking data for pagination/performance testing) where speed matters and realism through the API layer isn't the point being tested. Use API-based seeding (see [API Test Data Management](../04-api-testing/api-test-data-management.md)) when the creation path itself needs to be realistic/tested, or for smaller, test-specific data tied to a single test's precondition.

## Example

A reset-and-seed script establishing a known baseline state for a test environment:

```sql
-- reset_test_data.sql
-- Run before each test cycle to guarantee a known, consistent starting state

BEGIN;

-- Clear existing test data (order matters due to foreign key constraints —
-- child tables before parent tables)
TRUNCATE order_items, orders, products, customers RESTART IDENTITY CASCADE;

-- Seed baseline reference data: products
INSERT INTO products (id, name, price, stock_count, category) VALUES
    (101, 'Wireless Mouse', 899, 50, 'electronics'),
    (102, 'Mechanical Keyboard', 4999, 20, 'electronics'),
    (103, 'USB-C Cable', 299, 200, 'electronics'),
    (104, 'Out of Stock Widget', 199, 0, 'electronics');  -- deliberately for negative-path tests

-- Seed baseline test customers, one per tier for tier-specific test scenarios
INSERT INTO customers (id, email, tier) VALUES
    (1, 'qa_standard@example.com', 'standard'),
    (2, 'qa_premium@example.com', 'premium'),
    (3, 'qa_admin@example.com', 'admin');

COMMIT;
```

A parameterized bulk-generation script for volume/performance testing scenarios (Python generating the SQL, since hand-writing hundreds of rows isn't practical):

```python
def generate_bulk_order_seed_sql(num_orders: int) -> str:
    lines = ["INSERT INTO orders (customer_id, status, total) VALUES"]
    values = []
    for i in range(num_orders):
        customer_id = (i % 3) + 1   # cycle through the 3 seeded customers
        status = ["pending", "paid", "shipped"][i % 3]
        total = round(100 + (i * 7.5), 2)
        values.append(f"({customer_id}, '{status}', {total})")
    lines.append(",\n".join(values) + ";")
    return "\n".join(lines)

# Writes a ready-to-run SQL file for seeding 500 orders, useful for
# pagination/performance testing that needs realistic data volume
with open("seed_bulk_orders.sql", "w") as f:
    f.write(generate_bulk_order_seed_sql(500))
```

Running the reset script as part of a CI pipeline step, before the test suite executes:

```bash
psql -h test-db.example.com -U test_admin_user -d test_orders_db -f reset_test_data.sql
```

## Production Considerations

- Run seed/reset scripts with a dedicated database role that has write access only to the test/staging database — never point a reset script at production, and consider environment-name safeguards in the script/pipeline itself to prevent an accidental catastrophic run against the wrong target.
- Keep seed scripts under version control alongside the test suite itself — they should evolve together (a new required field on `orders` needs the seed script updated in the same PR as any test relying on it).
- `TRUNCATE ... RESTART IDENTITY CASCADE` (or the equivalent in your database) resets auto-increment IDs and cascades to dependent tables — useful for a fully clean baseline, but be deliberate about it, since it also means any hardcoded IDs referenced elsewhere in tests need to stay in sync with the reset script's seeded IDs.

## Common Pitfalls

- Running a reset/seed script against the wrong environment due to a misconfigured connection string — this is a serious, potentially catastrophic mistake; environment safeguards (explicit environment name confirmation, restricted credentials) are not optional for these scripts.
- Letting seed scripts drift out of sync with the actual schema over time, causing them to fail or seed incomplete data after an unrelated schema migration (see [Database Migration Testing](./database-migration-testing.md)) — seed scripts need the same maintenance discipline as the tests that depend on them.
- Seeding unrealistic data (e.g., every product priced at exactly ₹100) that doesn't exercise real-world variation, weakening the tests that use it.
- Forgetting foreign key order when truncating/deleting — attempting to delete a parent table's rows before its child table's referencing rows causes a constraint violation, unless `CASCADE` is used deliberately.

## Interview Notes

- Be ready to explain why bulk SQL-based seeding is sometimes preferred over API-based seeding, and vice versa — the trade-off between speed/bulk convenience and realism/exercising the real creation path.
- Understand the importance of safeguarding reset scripts against accidental production execution — this is a practical, real-world safety consideration interviewers value seeing awareness of.
- Be able to describe how you'd generate a large volume of realistic test data programmatically (rather than hand-writing hundreds of `INSERT` statements) — a common practical question for performance/pagination testing scenarios.

## References

- [PostgreSQL Documentation — TRUNCATE](https://www.postgresql.org/docs/current/sql-truncate.html)
- [PostgreSQL Documentation — INSERT](https://www.postgresql.org/docs/current/sql-insert.html)