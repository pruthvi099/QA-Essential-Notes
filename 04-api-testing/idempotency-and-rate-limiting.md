# Idempotency & Rate Limiting

## What It Is

This note covers two related but distinct API reliability concerns: **idempotency** — ensuring repeated identical requests produce a consistent, safe result (introduced in [HTTP Methods & Status Codes](./http-methods-and-status-codes.md), covered here in more practical depth, including idempotency for non-idempotent-by-default methods like POST) — and **rate limiting** — how an API protects itself from excessive requests, and how to test that clients are correctly throttled and informed.

## Why It Matters

- Network retries are a normal, expected part of distributed systems — a client that doesn't receive a response (due to a timeout, not necessarily a real failure) often retries the same request. If that request isn't idempotent, a retry can cause duplicate side effects (double charges, duplicate orders) — a real, costly production bug class.
- POST is *not* idempotent by default (see [HTTP Methods & Status Codes](./http-methods-and-status-codes.md)), but many real-world APIs (especially payment/order creation) need idempotent behavior anyway — this is achieved via a client-supplied **idempotency key**, a pattern worth understanding and testing specifically.
- Rate limiting is a common production safeguard, and testing it correctly (verifying the right status code, headers, and reset behavior) ensures both that abuse is actually prevented and that legitimate clients get clear, actionable feedback rather than a confusing failure.

## How It Works

**Idempotency keys** — a client generates a unique key (often a UUID) and sends it with a request (commonly in an `Idempotency-Key` header). The server stores the result of the first request with that key; if the same key is sent again, the server returns the *original* result instead of processing the action a second time.

**Testing idempotency keys:**
1. Send a request with a given idempotency key → verify it succeeds and creates the resource.
2. Send the *exact same* request with the *same* key again → verify it returns the same result without creating a duplicate resource.
3. Send a *different* request body with the *same* key → behavior here is API-specific (some reject as a conflict, some ignore the new body and return the original result) — verify against the actual documented contract.

**Rate limiting** — APIs typically respond with `429 Too Many Requests` once a client exceeds a defined limit, often including headers like `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` to communicate the limit and when the client can retry.

**Testing rate limiting:**
1. Send requests up to the limit → verify all succeed.
2. Exceed the limit → verify a `429` response with appropriate headers.
3. Wait for the reset window (or use a test-specific reset mechanism) → verify requests succeed again.

## Example

**Python — testing an idempotency key pattern:**
```python
import uuid

def test_idempotency_key_prevents_duplicate_order(api_context):
    idempotency_key = str(uuid.uuid4())
    payload = {
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 1}],
    }

    first_response = api_context.post(
        "/api/orders",
        headers={"Idempotency-Key": idempotency_key},
        data=payload,
    )
    assert first_response.status == 201
    first_order_id = first_response.json()["id"]

    # Same key, same payload, sent again — simulating a client retry
    # after a network timeout on the FIRST request's response
    second_response = api_context.post(
        "/api/orders",
        headers={"Idempotency-Key": idempotency_key},
        data=payload,
    )
    assert second_response.status == 201  # or 200, depending on API design
    second_order_id = second_response.json()["id"]

    # The critical assertion: NO duplicate order was created
    assert first_order_id == second_order_id

def test_different_idempotency_keys_create_separate_orders(api_context):
    payload = {"customer_email": "test@example.com", "items": [{"product_id": 101, "qty": 1}]}

    response_a = api_context.post("/api/orders", headers={"Idempotency-Key": str(uuid.uuid4())}, data=payload)
    response_b = api_context.post("/api/orders", headers={"Idempotency-Key": str(uuid.uuid4())}, data=payload)

    # Different keys = genuinely different requests, so two orders IS correct
    assert response_a.json()["id"] != response_b.json()["id"]
```

**TypeScript — testing rate limiting:**
```typescript
import { test, expect } from '@playwright/test';

test('exceeding rate limit returns 429 with correct headers', async ({ request }) => {
  const limit = 10;
  let lastResponse;

  // Send requests up to and beyond the documented limit
  for (let i = 0; i < limit + 1; i++) {
    lastResponse = await request.get('/api/products');
  }

  expect(lastResponse!.status()).toBe(429);
  expect(lastResponse!.headers()['retry-after']).toBeDefined();

  const body = await lastResponse!.json();
  expect(body.error).toContain('rate limit');
});

test('requests succeed again after the rate limit window resets', async ({ request }) => {
  // Trigger the limit
  for (let i = 0; i < 11; i++) {
    await request.get('/api/products');
  }

  const retryAfterResponse = await request.get('/api/products');
  const retryAfterSeconds = parseInt(retryAfterResponse.headers()['retry-after'] ?? '1');

  // Wait for the documented reset window before retrying
  await new Promise(resolve => setTimeout(resolve, (retryAfterSeconds + 1) * 1000));

  const afterResetResponse = await request.get('/api/products');
  expect(afterResetResponse.status()).toBe(200);
});
```

## Production Considerations

- Idempotency keys are especially critical for payment and order-creation endpoints — a non-idempotent payment API under real-world network retry conditions is a direct path to double-charging customers, one of the most reputationally damaging bug classes an API can have.
- Rate-limit testing that actually waits out the full reset window (as shown above) makes tests slow — consider whether the API/test environment supports a shorter test-specific rate limit window, or mock the rate-limiting layer for faster, still-meaningful coverage.
- Document and test the specific behavior for "same idempotency key, different request body" — this is an ambiguous edge case where different APIs make different design choices (reject vs. ignore-and-return-original), and it's a common source of integration confusion for API consumers if untested and undocumented.

## Common Pitfalls

- Assuming POST endpoints don't need idempotency testing since POST "isn't idempotent by design" — this misses that many real APIs deliberately add idempotency via keys specifically because retries are a normal occurrence in distributed systems.
- Not testing rate-limit headers (`Retry-After`, `X-RateLimit-Remaining`) — testing only the 429 status code without the headers misses whether clients actually have the information they need to back off and retry correctly.
- Testing rate limiting by guessing the limit through trial and error instead of checking documented values first — this wastes requests and time, and risks tripping broader account/IP-level throttling unrelated to the specific test.
- Not testing the "same idempotency key, same body, sent again" case specifically — testing only unique-key scenarios never actually exercises the deduplication logic idempotency keys exist to provide.

## Interview Notes

- Be ready to explain what an idempotency key is and why POST endpoints — despite not being idempotent by HTTP specification — often need one in real-world API design.
- Understand the real-world scenario idempotency keys solve: a client retries after a timeout, not knowing whether the original request actually succeeded server-side.
- Be able to describe how you'd test rate limiting thoroughly — not just triggering a 429, but verifying the informational headers and the reset behavior over time.

## References

- [Stripe — Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [MDN — 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429)
- [RFC 6585 — Additional HTTP Status Codes (429)](https://datatracker.ietf.org/doc/html/rfc6585)