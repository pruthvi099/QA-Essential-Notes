# API-to-Database Validation

## What It Is

API-to-database validation is the practice of cross-referencing what an API returns against what's actually stored in the database — verifying the two are consistent. This is a deeper, more systematic version of [Database Validation from API Tests](../04-api-testing/database-validation-from-api-tests.md): rather than a one-off check on a single field, it's a structured comparison approach for catching drift between what an API claims and what's actually persisted.

## Why It Matters

- An API layer can transform, cache, or compute data on the way out — a mismatch between the API response and the underlying database state can indicate a serialization bug, a stale cache, or a computation error, each requiring a different fix.
- This is especially important for APIs backed by caching layers, read replicas, or CQRS-style architectures, where the API might legitimately read from a different data source than the one being written to — validation here can reveal replication lag or cache invalidation bugs that are otherwise very hard to reproduce.
- Interviewers ask about this specifically to assess whether a candidate thinks about data consistency across architectural layers, not just within a single layer in isolation — a common signal of experience with real, non-trivial production systems.

## How It Works

**The core pattern:** call the API, extract the relevant fields from the response, then query the database directly for the same record, and assert field-by-field equivalence (accounting for any legitimate, expected transformation — e.g., a computed `display_price` in the API that isn't stored as-is in the DB).

**What to specifically watch for:**
1. **Direct field mismatches** — a field value differs between API and DB with no legitimate transformation reason.
2. **Stale/cached data** — the API returns an old value while the DB has already been updated (or vice versa, depending on write-through vs. write-behind caching).
3. **Computed field correctness** — fields the API calculates on the fly (totals, formatted values) should be independently verifiable by recomputing from raw DB data.
4. **Timing/consistency windows** — especially relevant in eventually-consistent systems, where a brief mismatch immediately after a write may be expected, not a bug.

## Example

```python
def test_api_response_matches_database_state(api_context, db_connection):
    # Create via API
    create_response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 2}],
    })
    api_order = create_response.json()
    order_id = api_order["id"]

    # Fetch the same record directly from the database
    cursor = db_connection.cursor()
    cursor.execute(
        "SELECT customer_email, status, total FROM orders WHERE id = %s",
        (order_id,)
    )
    db_row = cursor.fetchone()
    db_customer_email, db_status, db_total = db_row

    # Cross-reference: these should match exactly, no transformation expected
    assert api_order["customer_email"] == db_customer_email
    assert api_order["status"] == db_status
    assert float(api_order["total"]) == float(db_total)

def test_api_computed_field_matches_recalculated_value(api_context, db_connection):
    response = api_context.get("/api/orders/501")
    api_order = response.json()

    # The API returns a "display_total" that includes formatted currency —
    # recompute the raw value from the DB and verify the underlying number matches
    cursor = db_connection.cursor()
    cursor.execute("""
        SELECT SUM(price * qty) FROM order_items WHERE order_id = %s
    """, (501,))
    recalculated_total = cursor.fetchone()[0]

    # Strip formatting from the API's display value before comparing
    api_raw_value = float(api_order["display_total"].replace("₹", "").replace(",", ""))
    assert api_raw_value == float(recalculated_total)

def test_cache_reflects_recent_write_within_acceptable_window(api_context, db_connection):
    import time

    # Direct DB write (simulating a change made through another system)
    cursor = db_connection.cursor()
    cursor.execute("UPDATE products SET stock_count = 50 WHERE id = 101")
    db_connection.commit()

    # Poll the API briefly, allowing for expected cache propagation delay,
    # rather than asserting immediate consistency in an eventually-consistent system
    max_wait_seconds = 5
    for _ in range(max_wait_seconds):
        response = api_context.get("/api/products/101")
        if response.json()["stock_count"] == 50:
            break
        time.sleep(1)
    else:
        assert False, "API did not reflect DB update within acceptable cache window"
```

## Production Considerations

- Before writing an exact-match assertion between API and DB values, understand the system's actual consistency model — a strictly consistent system should match immediately; an eventually consistent one needs a bounded wait/retry (as shown in the caching example) rather than an immediate assertion that will be inherently flaky.
- When a mismatch is found, the debugging path differs by cause: a genuine data bug needs an application fix; a cache/replication lag issue needs either a longer test wait or a review of the caching strategy's actual guarantees — correctly categorizing which one you're looking at saves significant investigation time.
- This validation pattern is especially valuable immediately after a migration or a significant backend refactor (see [Database Migration Testing](./database-migration-testing.md)) — cross-referencing API and DB state broadly across many records can catch systemic drift a single spot-check would miss.

## Common Pitfalls

- Asserting immediate, exact consistency in a system that's legitimately eventually consistent — this produces flaky tests that fail intermittently for reasons unrelated to any real bug, exactly the pattern discussed in [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md).
- Comparing a computed/formatted API field directly against a raw DB field without accounting for the expected transformation, producing a false-positive mismatch.
- Not distinguishing between "the API is wrong" and "the API is stale" when a mismatch is found — these have different root causes and different fixes, and conflating them wastes debugging time.
- Only testing this pattern once, on a single record, instead of as a repeatable check — systemic drift (many records slightly out of sync) is often more valuable to detect than a single record's correctness.

## Interview Notes

- Be ready to explain why an API response and its underlying database record might legitimately differ (caching, eventual consistency, computed fields) — and how you'd account for each when writing a cross-validation test.
- Understand the difference between testing for exact consistency (appropriate for strictly consistent systems) and testing for eventual consistency (appropriate for cached/replicated systems, requiring a bounded wait/retry).
- Be able to describe how you'd investigate a mismatch to determine whether it's a genuine data bug versus expected propagation delay — this shows practical debugging judgment, not just the ability to write the comparison query.

## References

- [PostgreSQL Documentation — Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- [AWS — Eventual Consistency](https://aws.amazon.com/builders-library/challenges-with-distributed-systems/)