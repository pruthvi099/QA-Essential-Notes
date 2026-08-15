# API Chaining & Workflow Testing

## What It Is

API chaining is testing a sequence of dependent requests where each response feeds into the next request — a realistic user or system workflow, rather than isolated, independent endpoint checks. Examples: create an order → get its ID → check its status → cancel it, or log in → get a user ID → fetch that user's specific data. This tests whether the API works correctly as a connected flow, not just as individually correct endpoints.

## Why It Matters

- Individually correct endpoints don't guarantee a correct workflow — an order creation endpoint and an order-status endpoint can each pass their own isolated tests while the actual multi-step flow between them breaks (e.g., a newly created order isn't immediately queryable due to a caching or replication delay).
- Real-world usage is almost always a workflow, not a single isolated call — testing chains is what actually validates the scenarios users and integrating systems depend on.
- This is a practical skill interviewers probe directly, since it demonstrates thinking about end-to-end correctness rather than just individual request/response pairs — a common interview prompt is "design a test for [multi-step business process]."

## How It Works

**The core pattern:** extract a value from one response (an ID, a token, a generated code) and use it as input to the next request, asserting correctness at each step along the way — not just at the final step.

**Key testing considerations specific to chains:**
- **State consistency** — does data created in step 1 actually appear correctly when queried in step 2? (Tests for replication lag, caching issues.)
- **Partial failure handling** — if step 3 of a 5-step chain fails, is the system left in a valid, recoverable state, or a corrupted intermediate one?
- **Order dependency** — does the API correctly enforce that steps must happen in the right sequence (e.g., you can't ship an order that was never paid)?

## Example

**Python — a realistic order lifecycle chain:**
```python
def test_order_lifecycle_workflow(api_context):
    # Step 1: Create an order
    create_response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 2}],
    })
    assert create_response.status == 201
    order_id = create_response.json()["id"]
    assert create_response.json()["status"] == "pending"

    # Step 2: Verify the order is immediately queryable
    # (catches state-consistency bugs between write and read paths)
    get_response = api_context.get(f"/api/orders/{order_id}")
    assert get_response.status == 200
    assert get_response.json()["status"] == "pending"

    # Step 3: Process payment for the order
    payment_response = api_context.post(f"/api/orders/{order_id}/pay", data={
        "payment_method": "card",
        "card_token": "test-card-token",
    })
    assert payment_response.status == 200

    # Step 4: Verify status transitioned correctly as a RESULT of step 3
    status_after_payment = api_context.get(f"/api/orders/{order_id}")
    assert status_after_payment.json()["status"] == "paid"

    # Step 5: Attempt to ship — should only succeed because payment happened
    ship_response = api_context.post(f"/api/orders/{order_id}/ship")
    assert ship_response.status == 200
    assert api_context.get(f"/api/orders/{order_id}").json()["status"] == "shipped"

def test_cannot_ship_unpaid_order(api_context):
    # Negative chain test: verify the API enforces correct SEQUENCE
    create_response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 1}],
    })
    order_id = create_response.json()["id"]

    # Attempting to skip the payment step entirely
    ship_response = api_context.post(f"/api/orders/{order_id}/ship")
    assert ship_response.status == 409   # Conflict — invalid state transition
    assert api_context.get(f"/api/orders/{order_id}").json()["status"] == "pending"
```

**TypeScript — a chained login → fetch-user-specific-data flow:**
```typescript
import { test, expect } from '@playwright/test';

test('login then fetch own order history', async ({ request }) => {
  const loginResponse = await request.post('/api/login', {
    data: { email: 'test@example.com', password: 'Pass@123' },
  });
  expect(loginResponse.status()).toBe(200);
  const { access_token, user_id } = await loginResponse.json();

  const ordersResponse = await request.get(`/api/users/${user_id}/orders`, {
    headers: { Authorization: `Bearer ${access_token}` },
  });
  expect(ordersResponse.status()).toBe(200);

  const orders = await ordersResponse.json();
  // Every returned order should genuinely belong to THIS user —
  // a chained test can catch an authorization bug a single-endpoint
  // test in isolation might not surface as clearly
  for (const order of orders) {
    expect(order.customer_id).toBe(user_id);
  }
});
```

## Production Considerations

- Chained tests are inherently slower and more complex to debug than single-endpoint tests (a failure at step 4 might actually stem from bad data in step 1) — reserve them for genuinely multi-step business workflows, not as a default testing style for everything.
- When a chained test fails, log/assert intermediate state at each step (as shown above) rather than only asserting the final outcome — this makes it possible to pinpoint exactly which step introduced the problem instead of just knowing "the workflow is broken somewhere."
- Chained tests are a natural place to combine with API-based test data seeding (see [API Testing with Playwright](../02-automation-python-playwright/api-testing-with-playwright.md)) — each step's created data becomes the next step's precondition, keeping the whole chain self-contained and independent of other tests.

## Common Pitfalls

- Only asserting on the final step's result, missing exactly *where* in the chain a failure originated — this turns straightforward debugging into a much slower investigation.
- Not testing invalid sequencing (attempting to skip or reorder steps) — this is where real business-logic enforcement bugs live, and it's a distinctly different (and often overlooked) risk from simply testing the correct sequence.
- Building overly long, brittle chains (10+ dependent steps) where a single early failure cascades and obscures the actual test intent — breaking very long chains into smaller, more focused chained tests is usually clearer.
- Not accounting for realistic timing/consistency issues (e.g., asynchronous processing between steps) — a chain that assumes immediate consistency when the real system is eventually consistent can produce flaky, timing-dependent failures.

## Interview Notes

- Be ready to design a chained test for a realistic multi-step business process (e.g., "test the full checkout flow via API calls only") — a very common, practical interview exercise.
- Understand why testing invalid sequencing (skipping/reordering steps) is as important as testing the correct sequence — this is often the differentiator between a basic and a thorough answer.
- Be able to explain how you'd debug a failing chained test efficiently — asserting/logging intermediate state at each step, not just the final result.

## References

- [ISTQB Foundation Level Syllabus — Integration Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)