# Database Migration Testing

## What It Is

Database migration testing verifies that a schema change (adding/removing/renaming a column, changing a data type, adding a constraint) applies correctly without data loss, corruption, or breaking application compatibility. This includes both testing the migration script itself and verifying the application still functions correctly against the new schema — a distinct concern from the routine data-integrity checks earlier in this folder, since migrations are a one-time, higher-risk event applied to existing production data.

## Why It Matters

- A migration bug can be uniquely damaging compared to other bug classes — unlike an application bug that's contained until fixed, a bad migration can silently corrupt or lose existing production data the moment it runs, sometimes irreversibly.
- Migrations often need to run against real-world data volumes and edge cases that don't exist in a small test dataset (unexpected NULLs, legacy data formats from years of accumulated records) — this makes migration testing a place where realistic test data genuinely matters more than in most other testing contexts.
- This is a genuinely advanced, often underappreciated SDET responsibility — being able to speak to migration testing specifically (not just "I write SQL queries") signals experience with real production system lifecycle concerns, not just isolated feature testing.

## How It Works

**What to verify for any migration:**

1. **Data preservation** — existing data survives the migration correctly (no unexpected NULLs, no truncated values, no lost rows).
2. **Backward compatibility during rollout** — if deployed with zero downtime, does the application (potentially running old and new code simultaneously during a rolling deploy) work correctly against both the old and new schema during the transition window?
3. **Rollback safety** — if the migration needs to be reverted, does the rollback script correctly restore the previous state without further data loss?
4. **Constraint/default value correctness** — new NOT NULL columns need a sensible default or backfill for existing rows; new constraints must not reject existing, previously-valid data.
5. **Performance on realistic data volume** — a migration that runs instantly on 100 test rows might take hours (and lock the table) against millions of production rows — this needs to be tested against a realistically-sized dataset, not just a small sample.

## Example

Testing a migration that adds a new `NOT NULL` column with a default value:

```sql
-- The migration being tested
ALTER TABLE orders ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD';
```

```python
def test_migration_backfills_existing_rows_correctly(db_connection):
    cursor = db_connection.cursor()

    # Before running the migration in this test environment, seed
    # some pre-existing rows to simulate real production data
    cursor.execute("INSERT INTO orders (customer_id, total) VALUES (1, 500), (2, 750)")
    db_connection.commit()

    # Run the migration
    cursor.execute("ALTER TABLE orders ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD'")
    db_connection.commit()

    # Verify existing rows were backfilled with the default, not left NULL
    # or the migration failing outright due to the NOT NULL constraint
    cursor.execute("SELECT id, currency FROM orders")
    rows = cursor.fetchall()
    for order_id, currency in rows:
        assert currency == 'USD', f"Order {order_id} was not correctly backfilled"

def test_migration_does_not_break_existing_application_queries(db_connection, api_context):
    # After the migration, existing application code (which doesn't
    # yet know about the new column) should still function correctly —
    # this simulates the rolling-deploy compatibility window
    cursor = db_connection.cursor()
    cursor.execute("ALTER TABLE orders ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD'")
    db_connection.commit()

    # Existing, unmodified application flow should still work
    response = api_context.get("/api/orders/1")
    assert response.status == 200

def test_migration_rollback_restores_previous_state(db_connection):
    cursor = db_connection.cursor()

    cursor.execute("ALTER TABLE orders ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD'")
    db_connection.commit()

    # Run the corresponding rollback script
    cursor.execute("ALTER TABLE orders DROP COLUMN currency")
    db_connection.commit()

    # Verify the schema is genuinely back to its prior state
    cursor.execute("""
        SELECT column_name FROM information_schema.columns
        WHERE table_name = 'orders' AND column_name = 'currency'
    """)
    assert cursor.fetchone() is None, "Rollback did not remove the migrated column"
```

## Production Considerations

- Test migrations against a realistic *copy* of production-scale data volume, not just a handful of test rows — a migration that adds an index or a NOT NULL constraint can behave very differently (in terms of lock duration and performance) at real scale versus a small test dataset, and this difference is exactly what causes production migration incidents.
- For zero-downtime deployments, explicitly test the compatibility window where old application code runs against the new schema (and sometimes vice versa, if rolling back) — this is a distinct, often-overlooked risk area beyond just "does the migration itself succeed."
- Always have and test a rollback plan before running a migration against production — an untested rollback script is a false sense of safety, since migration rollbacks can have their own bugs (data loss during rollback is a real, serious risk).

## Common Pitfalls

- Testing a migration only against a small, clean test dataset that doesn't reflect the messiness of real production data (unexpected NULLs, legacy formats, orphaned records) — migrations often fail specifically on the edge cases small test data doesn't contain.
- Adding a `NOT NULL` constraint without a default value or backfill plan for existing rows, causing the migration to fail outright (or worse, silently succeed while leaving unexpected behavior) against non-empty tables.
- Not testing the application's behavior during the rolling-deploy compatibility window, only testing "before" and "after" states — the messy in-between period (old code + new schema, or new code + old schema) is where many real migration-related production incidents actually occur.
- Assuming a migration is safe because it worked in a small test environment, without validating performance/locking behavior against realistic data volume.

## Interview Notes

- Be ready to describe what you'd test before approving a schema migration for production — data preservation, backward compatibility, rollback safety, and performance at scale are the core categories interviewers expect.
- Understand why testing against realistic data volume (not just a small test dataset) matters specifically for migrations, more so than for most other kinds of testing.
- Be able to describe the "compatibility window" risk in a zero-downtime rolling deployment, and how you'd test for it — this is a more advanced point that distinguishes deeper production-systems experience.

## References

- [PostgreSQL Documentation — ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html)
- [Martin Fowler — Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)