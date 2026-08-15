# API Test Data Management

## What It Is

API test data management covers how test data is created, isolated, and cleaned up specifically for API-level tests — extending the general principles from [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) into the automated, programmatic context of API testing, where data setup/teardown can (and should) happen via API calls themselves rather than manual steps or UI interaction.

## Why It Matters

- API tests run far more frequently than manual test cycles (every commit, every PR) — poor data isolation at this scale doesn't just cause occasional inconsistency, it causes constant, hard-to-diagnose flakiness across a suite run many times a day.
- Because API tests are fast, they're often run in parallel (see [Parallel Execution](../02-automation-python-playwright/parallel-execution.md)) — this makes data collision between concurrently running tests a much more immediate, common risk than in slower, sequential manual testing.
- Test data setup via API calls (rather than direct database manipulation or UI steps) is both faster and more realistic — it exercises the same code paths real clients use, catching bugs that direct DB seeding would bypass entirely.

## How It Works

**Core strategies:**

1. **Setup via API, not the UI or direct DB access** — creating a test order via a `POST /api/orders` call is both fast and realistic (see [API Testing with Playwright](../02-automation-python-playwright/api-testing-with-playwright.md)); it also means the test data creation path is itself continuously tested as a side effect.
2. **Unique data per test** — generate unique identifiers (UUIDs, timestamps) for emails, usernames, and other unique fields, so parallel/concurrent test runs never collide.
3. **Teardown after the test** — delete created resources (via API calls) after each test, either in the test itself or a fixture's teardown phase, to avoid data accumulation across many CI runs over time.
4. **Isolated test accounts/tenants** — for multi-tenant systems, using a dedicated test tenant (rather than a shared one) further isolates test data from both production data and other tests.

**Setup vs. teardown trade-off:** always cleaning up after every test keeps the environment tidy but adds overhead and risk (a failed teardown leaves orphaned data); some teams instead rely entirely on unique-per-test data and periodic bulk environment resets, skipping per-test teardown for speed. Either is valid — the key requirement is that the *chosen* strategy is applied consistently.

## Example

**Python — a fixture handling both setup and guaranteed teardown via API:**
```python
import pytest
import uuid

@pytest.fixture
def test_order(api_context):
    unique_email = f"test_{uuid.uuid4().hex[:8]}@example.com"

    # Setup: create via API, not the UI or direct DB access
    create_response = api_context.post("/api/orders", data={
        "customer_email": unique_email,
        "items": [{"product_id": 101, "qty": 1}],
    })
    order = create_response.json()

    yield order   # test runs here

    # Teardown: always runs, even if the test itself fails/errors
    api_context.delete(f"/api/orders/{order['id']}")

def test_order_can_be_cancelled(api_context, test_order):
    response = api_context.post(f"/api/orders/{test_order['id']}/cancel")
    assert response.status == 200
    assert api_context.get(f"/api/orders/{test_order['id']}").json()["status"] == "cancelled"
    # No manual cleanup needed here — the fixture's teardown handles it
    # regardless of whether this test passes or fails
```

**TypeScript — the same pattern using a custom fixture:**
```typescript
import { test as base, expect } from '@playwright/test';

type Order = { id: number; status: string };

const test = base.extend<{ testOrder: Order }>({
  testOrder: async ({ request }, use) => {
    const uniqueEmail = `test_${Date.now()}_${Math.random().toString(36).slice(2, 8)}@example.com`;

    const createResponse = await request.post('/api/orders', {
      data: { customer_email: uniqueEmail, items: [{ productId: 101, qty: 1 }] },
    });
    const order = await createResponse.json();

    await use(order);   // test runs here

    // Teardown always runs after the test, pass or fail
    await request.delete(`/api/orders/${order.id}`);
  },
});

test('order can be cancelled', async ({ request, testOrder }) => {
  const response = await request.post(`/api/orders/${testOrder.id}/cancel`);
  expect(response.status()).toBe(200);
});
```

## Production Considerations

- Fixture-based setup/teardown (as shown) is more reliable than manual cleanup calls scattered inside individual tests — a fixture's teardown runs even if the test body fails or throws, whereas manual cleanup code placed after assertions in the test body gets skipped entirely if an earlier assertion fails.
- For very high-volume CI usage, consider periodic bulk cleanup (a scheduled job that purges old test data by a naming convention or age) as a safety net — even well-implemented per-test teardown can leave orphaned data behind occasionally (a CI runner crash mid-test, for example).
- Namespace or tag test-created data clearly (a consistent email domain like `@test.example.com`, a `source: automated-test` field) — this makes bulk identification and cleanup straightforward, and prevents test data from ever being mistaken for real data in shared/staging environments.

## Common Pitfalls

- Hardcoding test data (a fixed email, a fixed username) instead of generating unique values per test run — this is the single most common cause of API test flakiness under parallel execution, as covered in [Parallel Execution](../02-automation-python-playwright/parallel-execution.md).
- Relying on manual cleanup code inside the test body instead of fixture-based teardown — cleanup code placed after assertions gets skipped if an assertion fails, silently leaving orphaned data behind on every failing test.
- Seeding test data via direct database manipulation instead of real API calls — this is faster in the short term but bypasses the actual code path real clients use, missing bugs in the creation logic itself and creating data that may not perfectly match what the API would have produced.
- Not having any cleanup strategy at all, letting test data accumulate indefinitely in a shared environment — this eventually degrades environment performance and can make manual/exploratory testing in the same environment confusing, since it's cluttered with old automated test artifacts.

## Interview Notes

- Be ready to explain why creating test data via API calls (not direct DB manipulation) is generally preferred — it exercises real code paths and stays realistic to actual client behavior.
- Understand why fixture-based teardown is more reliable than manual cleanup code in the test body — specifically, that it still runs even when the test fails.
- Be able to describe how you'd prevent data collisions in a parallelized API test suite — unique per-test data generation is the expected, practical answer.

## References

- [Playwright — Fixtures (Node.js)](https://playwright.dev/docs/test-fixtures)
- [Pytest — Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)