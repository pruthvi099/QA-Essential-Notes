# Transaction & Consistency Testing

## What It Is

Transaction testing verifies that database operations correctly follow **ACID properties** — Atomicity, Consistency, Isolation, Durability — under real-world conditions like partial failures, concurrent access, and rollback scenarios. This is distinct from the single-request validation covered so far in this folder — it specifically targets what happens when things go wrong *mid-operation*, or when multiple operations happen at once.

## Why It Matters

- Multi-step database operations (deduct inventory + create an order + charge a payment) need to either fully succeed or fully fail together — a partial failure that leaves the system in an inconsistent state (inventory deducted, but no order created) is a serious, often hard-to-detect production bug class.
- Race conditions from concurrent access are notoriously difficult to catch through normal sequential testing — two simultaneous requests can interact in ways a single-threaded test suite never naturally exercises, requiring deliberately concurrent test design.
- This is genuinely advanced material that distinguishes SDETs who understand systems-level correctness from those who only test individual requests in isolation — it's frequently referenced in senior SDET interviews specifically for that reason.

## How It Works

**ACID properties, and what to test for each:**

- **Atomicity** — an operation's steps all succeed or all fail together. Test: deliberately trigger a failure partway through a multi-step operation (e.g., simulate a payment failure after inventory was already decremented) and verify the *entire* operation rolled back, not just the failed step.
- **Consistency** — the database moves from one valid state to another, never violating defined rules (constraints, business rules). Test: verify constraint violations are correctly rejected (e.g., a foreign key constraint prevents an order from referencing a non-existent customer).
- **Isolation** — concurrent transactions don't interfere with each other incorrectly. Test: run two operations concurrently that touch the same data (e.g., two simultaneous purchases of the last item in stock) and verify the correct one wins without both succeeding and overselling.
- **Durability** — once committed, data survives (isn't lost on a crash/restart). Harder to test directly in typical QA scope; usually verified through infrastructure/DR testing rather than application-level tests.

## Example

**Testing atomicity — a simulated partial failure should roll back completely:**
```python
def test_order_creation_rolls_back_on_payment_failure(api_context, db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT stock_count FROM products WHERE id = 101")
    stock_before = cursor.fetchone()[0]

    # Trigger an order using a payment method KNOWN to fail
    # (a test-mode "always declines" card token)
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 1}],
        "payment_token": "test-always-declines-token",
    })
    assert response.status == 402  # Payment Required / declined

    # Critical assertion: inventory should NOT have been decremented,
    # since the overall order operation failed and should have rolled back
    cursor.execute("SELECT stock_count FROM products WHERE id = 101")
    stock_after = cursor.fetchone()[0]
    assert stock_after == stock_before, (
        "Inventory was decremented despite the order failing — "
        "the operation did not roll back atomically"
    )

    # No order record should exist either, for the same reason
    cursor.execute(
        "SELECT COUNT(*) FROM orders WHERE customer_email = %s AND status = 'pending'",
        ("test@example.com",)
    )
    assert cursor.fetchone()[0] == 0
```

**Testing isolation — a concurrency/race condition test for the "last item in stock" scenario:**
```python
import threading

def test_concurrent_purchases_do_not_oversell_last_item(api_context, db_connection):
    # Set stock to exactly 1
    cursor = db_connection.cursor()
    cursor.execute("UPDATE products SET stock_count = 1 WHERE id = 101")
    db_connection.commit()

    results = []

    def attempt_purchase():
        response = api_context.post("/api/orders", data={
            "customer_email": f"test_{threading.get_ident()}@example.com",
            "items": [{"product_id": 101, "qty": 1}],
        })
        results.append(response.status)

    # Fire two purchase attempts at effectively the same time
    thread_a = threading.Thread(target=attempt_purchase)
    thread_b = threading.Thread(target=attempt_purchase)
    thread_a.start()
    thread_b.start()
    thread_a.join()
    thread_b.join()

    # Exactly ONE should succeed (201), the other should be correctly
    # rejected (e.g., 409 Conflict) — both succeeding means the system
    # oversold, a real isolation/concurrency bug
    success_count = results.count(201)
    assert success_count == 1, f"Expected exactly 1 success, got {success_count}: {results}"

    cursor.execute("SELECT stock_count FROM products WHERE id = 101")
    assert cursor.fetchone()[0] == 0
```

## Production Considerations

- Concurrency tests are inherently less deterministic than sequential tests — timing can vary between runs, so these tests may need to run multiple iterations or use synchronization primitives (barriers) to maximize the chance of triggering the actual race condition being tested.
- Atomicity/rollback testing typically requires either a genuine failure injection point (a test-mode "always fails" payment token, as shown) or the ability to simulate a mid-transaction failure — coordinate with developers to ensure such test hooks exist deliberately, rather than trying to awkwardly force real failures.
- These tests are expensive to write and maintain relative to simpler functional tests — reserve them for genuinely high-risk operations (payments, inventory, anything with real financial/business consequences) rather than applying transaction/concurrency testing universally.

## Common Pitfalls

- Only testing the happy path of multi-step operations, never verifying rollback behavior when a step fails partway through — this is exactly where atomicity bugs hide, since they're invisible when everything succeeds.
- Writing concurrency tests that aren't actually concurrent (e.g., using sequential requests with a manual delay instead of true parallel execution) — this doesn't reliably exercise the race condition being tested for.
- Assuming database-level transactions automatically make an entire *application-level* operation atomic — if a multi-step business operation spans multiple separate database transactions (or worse, multiple services), true atomicity requires deliberate application-level design (like a saga pattern), not just wrapping one query in `BEGIN`/`COMMIT`.
- Treating a flaky concurrency test result as "just flakiness" instead of investigating whether it's revealing a genuine, intermittent race condition — see the same diagnostic discipline in [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md).

## Interview Notes

- Be ready to explain ACID properties from memory, with a concrete testing approach for at least Atomicity and Isolation specifically — these two are the most commonly probed in interviews, since Consistency and Durability are often tested more indirectly.
- Understand how to design a genuine concurrency test (parallel threads/requests hitting the same resource) versus a sequential test that only simulates concurrency — a common, specific follow-up question.
- Be able to describe a real "oversold last item" or similar race-condition scenario and how you'd test for it — this is a frequently used, concrete example in senior SDET interviews.

## References

- [PostgreSQL Documentation — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [ACID (Wikipedia, as a starting orientation — verify specifics against your database's own documentation)](https://en.wikipedia.org/wiki/ACID)