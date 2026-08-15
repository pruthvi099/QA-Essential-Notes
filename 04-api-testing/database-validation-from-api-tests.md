# Database Validation from API Tests

## What It Is

Database validation from API tests means verifying that an API action didn't just return a successful response, but actually persisted the correct data at the database layer. This closes a real gap that response-only assertions can't cover: an API can return `201 Created` with a plausible-looking body while the actual write to the database fails, writes incorrect data, or writes to the wrong table/record — none of which a response-only test would ever catch.

## Why It Matters

- A successful-looking API response doesn't guarantee successful persistence — bugs in transaction handling, async write queues, or ORM logic can cause a response to claim success while the actual database state is wrong or unchanged, a gap only a direct DB check can catch.
- This is the practical bridge between API testing (this folder) and [05-sql-database-testing](../05-sql-database-testing/) — many real-world API test suites include targeted DB assertions specifically for high-risk write operations, without needing every test to duplicate this.
- Interviewers ask about this specifically to see whether a candidate tests only the "contract" (the response) or also verifies the actual "effect" (the real system state) — the latter shows a deeper, more skeptical testing mindset.

## How It Works

**The core pattern:** perform an API action, then query the database directly to confirm the expected state actually exists — not just that the API said it would.

**When this is worth doing (not every test needs it):**
- High-risk write operations — payments, order creation, account changes — where a false-positive "success" response would be costly.
- Verifying complex side effects the response body doesn't fully expose (e.g., an inventory count decremented as a side effect of an order, not returned in the order response itself).
- Investigating a suspected bug where the API claims success but downstream behavior suggests otherwise.

**When it's usually unnecessary:** most functional API tests are well-served by response validation alone (see [Request & Response Validation](./request-response-validation.md)) — reaching for direct DB checks on every test adds test complexity and a dependency on DB access/schema knowledge that isn't always worth it.

## Example

**Python — verifying an order was actually persisted correctly, beyond just the API response:**
```python
import psycopg2
import pytest

@pytest.fixture
def db_connection():
    conn = psycopg2.connect(
        host="test-db.example.com",
        dbname="test_orders_db",
        user="test_readonly_user",
        password="from-env-var",
    )
    yield conn
    conn.close()

def test_order_creation_persists_correctly(api_context, db_connection):
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 2}],
    })
    assert response.status == 201
    order_id = response.json()["id"]

    # Verify the ACTUAL database state, not just the API's claim
    cursor = db_connection.cursor()
    cursor.execute(
        "SELECT customer_email, status FROM orders WHERE id = %s", (order_id,)
    )
    row = cursor.fetchone()

    assert row is not None, "Order was not found in the database at all"
    assert row[0] == "test@example.com"
    assert row[1] == "pending"

    cursor.execute(
        "SELECT product_id, qty FROM order_items WHERE order_id = %s", (order_id,)
    )
    items = cursor.fetchall()
    assert len(items) == 1
    assert items[0] == (101, 2)

def test_order_creation_decrements_inventory(api_context, db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT stock_count FROM products WHERE id = 101")
    stock_before = cursor.fetchone()[0]

    api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 3}],
    })

    # Inventory decrement is a SIDE EFFECT not visible in the order
    # response body at all — only a direct DB check can verify it
    cursor.execute("SELECT stock_count FROM products WHERE id = 101")
    stock_after = cursor.fetchone()[0]

    assert stock_after == stock_before - 3
```

## Production Considerations

- Always use a dedicated, read-only database user/role for test verification queries — tests should never have write access to the database beyond what the API itself performs, to avoid tests accidentally corrupting data through direct manipulation.
- Reserve direct DB validation for genuinely high-risk operations or side effects not observable through the API/response alone — overusing it across every test tightly couples the test suite to the database schema, making routine schema changes break far more tests than necessary.
- If direct DB access isn't available in a given test environment (common in fully managed/black-box API testing scenarios), an internal verification endpoint or admin API can sometimes serve a similar purpose — the principle (verify real state, not just the response claim) matters more than the specific mechanism.

## Common Pitfalls

- Trusting a successful API response as sufficient proof of correct persistence for high-risk operations, missing a class of bugs where the response and the actual database state diverge.
- Coupling too many tests directly to database schema details — when the schema legitimately changes, an excessive number of DB-validating tests break simultaneously, creating unnecessary maintenance overhead.
- Using a database user with write access for test verification, risking accidental test-induced data corruption if a query is malformed.
- Not accounting for asynchronous persistence (e.g., a write processed via a queue slightly after the API responds) — querying the database immediately after the API call, without any wait/retry allowance, can produce a flaky false failure if the write hasn't completed yet.

## Interview Notes

- Be ready to explain a concrete scenario where a response-only test would miss a real bug that a DB-level check would catch (e.g., an inventory decrement not reflected in the order response).
- Understand the trade-off: DB validation gives stronger correctness guarantees but couples tests to schema and adds complexity — be able to articulate when the trade-off is worth it (high-risk operations) versus not (routine functional checks).
- Be able to describe why test database access should be read-only, and what risk write access would introduce.

## References

- [ISTQB Foundation Level Syllabus — Integration Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)