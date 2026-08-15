# Request & Response Validation

## What It Is

Request and response validation is the systematic practice of verifying every meaningful part of an HTTP exchange — not just the status code, but headers, body structure, field values, and types. A test that only checks `status == 200` verifies the API responded, not that it responded *correctly*. This note covers what a thorough API test actually validates and why each layer matters.

## Why It Matters

- Status-code-only assertions are a common, dangerous shortcut — an API can return 200 with a completely wrong or malformed body, and a status-only test would pass while a real bug ships.
- Different layers of a response catch different classes of bugs: headers catch caching/security misconfigurations, body structure catches contract breaks, field values catch business logic errors — skipping any layer creates a specific, predictable blind spot.
- This is one of the most practically tested SDET skills — being handed an API response and asked "what would you validate here?" is a very common interview exercise, and a thorough, layered answer signals real experience over surface familiarity.

## How It Works

**What a thorough API test validates, layer by layer:**

1. **Status code** — the correct code for the specific scenario (not just "success," but the *right* success/error code — see [HTTP Methods & Status Codes](./http-methods-and-status-codes.md))
2. **Headers** — `Content-Type` matches expected format, caching headers are correct, security headers are present where required (e.g., `X-Content-Type-Options`)
3. **Response body structure** — required fields are present, no unexpected fields leak sensitive data, nested objects have the right shape
4. **Field values and types** — a field that should be a number isn't returned as a string, an email field is actually a valid email format, an enum field only contains allowed values
5. **Business logic correctness** — the actual values are *right*, not just present and correctly typed (e.g., a calculated total actually equals the sum of line items)

**Request validation** matters too, though less commonly tested directly — verifying the API correctly rejects malformed requests (see [Negative Testing for APIs](./negative-testing-for-apis.md)) is the complementary check to response validation.

## Example

A layered validation approach for a single API response, showing what "thorough" actually looks like in practice:

```python
def test_get_order_full_validation(api_context):
    response = api_context.get("/api/orders/501")

    # Layer 1: Status code — the specific expected code
    assert response.status == 200

    # Layer 2: Headers
    assert response.headers["content-type"].startswith("application/json")

    body = response.json()

    # Layer 3: Structure — required fields present
    required_fields = {"id", "customer_email", "items", "total", "status", "created_at"}
    assert required_fields.issubset(body.keys())

    # Layer 3 (continued): No unexpected sensitive data leaked
    assert "customer_password_hash" not in body
    assert "internal_notes" not in body

    # Layer 4: Field types and formats
    assert isinstance(body["id"], int)
    assert isinstance(body["total"], (int, float))
    assert "@" in body["customer_email"]
    assert body["status"] in ["pending", "shipped", "delivered", "cancelled"]

    # Layer 5: Business logic correctness — not just present, but RIGHT
    calculated_total = sum(item["price"] * item["qty"] for item in body["items"])
    assert body["total"] == calculated_total
```

A version that stops at status-code-only, shown for contrast — this would pass even if every layer above it were broken:

```python
def test_get_order_shallow(api_context):
    response = api_context.get("/api/orders/501")
    assert response.status == 200
    # Nothing else checked — a malformed body, leaked sensitive field,
    # or wrong total would all silently pass this test
```

## Production Considerations

- Combine layered manual assertions (as shown above) with schema validation (see [JSON Schema Validation](./json-schema-validation.md)) for structure/type checks — schema validation scales better for structural correctness across many fields, while manual assertions remain necessary for business-logic-specific checks (like the total calculation above) that a generic schema can't express.
- Explicitly test for sensitive data leakage in responses (password hashes, internal IDs, other users' data) — this is both a functional and a security concern (see [API Security Testing Basics](./api-security-testing-basics.md)), and status-only tests structurally cannot catch it.
- When an API contract changes (a field renamed, a type changed), thorough response validation is what catches it immediately in CI — shallow tests let contract breaks reach consumers (frontend, other services) silently.

## Common Pitfalls

- Asserting only on status code, missing the majority of real bugs that a "successful" response can still contain.
- Checking that a field exists without checking its type or format — a `price` field returned as `"999"` (string) instead of `999` (number) can break downstream consumers despite passing an existence-only check.
- Not checking for unexpected or sensitive fields in the response — over-fetching/data-leakage bugs are common and easy to catch with an explicit check, but easy to miss if validation only confirms required fields are present.
- Validating structure but never business logic — confirming a `total` field exists and is a number doesn't catch it being calculated incorrectly.

## Interview Notes

- Be ready to walk through, layer by layer, everything you'd validate given a sample API response — status, headers, structure, types, business logic — this is a very common practical exercise.
- Understand why status-code-only testing is a significant, common testing gap, with a concrete example of a bug it would miss.
- Be able to explain the difference between structural validation (a field exists and has the right type) and business logic validation (the field's actual value is correct) — and why both are needed.

## References

- [MDN — HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)