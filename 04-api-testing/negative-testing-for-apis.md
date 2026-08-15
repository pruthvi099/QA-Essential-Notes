# Negative Testing for APIs

## What It Is

Negative testing verifies that an API correctly *rejects* invalid input, rather than only confirming it accepts valid input. This means deliberately sending malformed requests — missing required fields, wrong data types, out-of-range values, invalid formats — and asserting the API responds with the correct error, not a crash, not a silent acceptance of bad data, and not a misleading success response.

## Why It Matters

- Happy-path-only API testing leaves an entire, often larger class of real bugs uncovered — APIs frequently handle valid input correctly (since it's the primary use case developers build and test against) while mishandling invalid input in ways that cause data corruption, crashes, or security issues.
- A poorly validated API that "succeeds" on bad input (e.g., accepting a negative quantity, or silently truncating an oversized string instead of rejecting it) can corrupt downstream data in ways that are much harder to trace back to the root cause than an immediate, clear rejection would have been.
- This directly extends [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md) (Equivalence Partitioning, Boundary Value Analysis) applied specifically at the API layer — negative testing is where those techniques produce the most concrete, high-value test cases.

## How It Works

**Categories of negative test cases:**

1. **Missing required fields** — omit each required field one at a time, verify a clear 400/422 response, not a 500.
2. **Wrong data types** — send a string where a number is expected, a number where a boolean is expected.
3. **Out-of-range / invalid values** — negative quantities, zero where a positive number is required, an enum value outside the allowed set.
4. **Invalid formats** — malformed email addresses, invalid date formats, improperly structured IDs.
5. **Oversized input** — extremely long strings, deeply nested JSON, unusually large arrays — testing whether the API enforces reasonable limits rather than accepting anything.
6. **Malformed request body** — invalid JSON syntax entirely, wrong `Content-Type` header.
7. **Unexpected extra fields** — fields not defined in the API contract, verifying the API ignores or rejects them appropriately rather than processing them unexpectedly.

**What a correct negative response looks like:** a specific 4xx status code (not a generic 500, which indicates the *server* crashed rather than correctly validating input), and ideally a clear error message identifying which field/rule failed — useful for both real API consumers and test debugging.

## Example

A systematic negative test suite for an order-creation endpoint, applying Equivalence Partitioning and Boundary Value Analysis directly:

```python
import pytest

def test_missing_required_field_items(api_context):
    response = api_context.post("/api/orders", data={
        # "items" field omitted entirely
        "customer_email": "test@example.com",
    })
    assert response.status == 400
    assert "items" in response.json()["error"].lower()

@pytest.mark.parametrize("qty,expected_status", [
    (-1, 400),    # negative quantity — invalid
    (0, 400),     # zero quantity — invalid
    (1, 201),     # minimum valid quantity — boundary
    (100, 201),   # reasonable valid quantity
])
def test_quantity_boundary_validation(api_context, qty, expected_status):
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": qty}],
    })
    assert response.status == expected_status

def test_wrong_type_for_quantity(api_context):
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": "two"}],   # string instead of int
    })
    assert response.status == 400

def test_invalid_email_format(api_context):
    response = api_context.post("/api/orders", data={
        "customer_email": "not-a-valid-email",
        "items": [{"product_id": 101, "qty": 1}],
    })
    assert response.status == 400
    assert "email" in response.json()["error"].lower()

def test_malformed_json_body(api_context):
    response = api_context.post(
        "/api/orders",
        headers={"Content-Type": "application/json"},
        data="{ this is not valid json",
    )
    # Should be a clear 400, NOT a 500 — a 500 here indicates the
    # server crashed on malformed input rather than validating it
    assert response.status == 400

def test_oversized_input_rejected(api_context):
    huge_string = "a" * 100_000
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 1, "notes": huge_string}],
    })
    assert response.status == 400   # should reject, not silently truncate or 500

def test_unexpected_extra_field_handled_safely(api_context):
    response = api_context.post("/api/orders", data={
        "customer_email": "test@example.com",
        "items": [{"product_id": 101, "qty": 1}],
        "is_admin_override": True,   # field the client should NOT be able to set
    })
    # The API should either ignore this field or reject it —
    # it must NOT silently grant admin-level behavior from client input
    assert response.status in [201, 400]
    if response.status == 201:
        order = response.json()
        assert "is_admin_override" not in order or order.get("admin_applied") is not True
```

## Production Considerations

- Treat a 500 (server error) response to *any* malformed input as a genuine defect worth filing immediately — a well-built API should never crash on invalid input; it should validate and return a clear 4xx instead. A 500 here often indicates a missing validation layer, which can also be a security gap.
- The "unexpected extra field" test category is a specific, practical check against mass-assignment vulnerabilities — where a client sneaks in a field like `is_admin: true` that the API naively applies without checking if the client should be allowed to set it. This is a real, common vulnerability class, not a theoretical edge case.
- Prioritize negative testing according to risk (see [Risk-Based Testing](../00-start-here/risk-based-testing.md)) — endpoints handling payments, permissions, or user-generated content deserve the deepest negative test coverage.

## Common Pitfalls

- Writing extensive happy-path coverage but only one or two token negative tests "to check the box" — negative testing needs the same systematic technique application (equivalence partitioning, boundaries) as positive testing, not an afterthought.
- Accepting a 500 response as "the test caught the bug" without filing it as a defect — a 500 on invalid input is itself the bug (missing validation), not just an inconvenient test result.
- Not testing the mass-assignment scenario (extra/unauthorized fields in the request) — this is a specific, real vulnerability class that's easy to test for once you know to look for it, but easy to miss otherwise.
- Only testing individually invalid fields, never combinations (e.g., multiple invalid fields at once) — while less critical than individual field validation, this can reveal error-handling bugs where multiple simultaneous validation failures produce a confusing or incomplete error response.

## Interview Notes

- Be ready to list the categories of negative test cases you'd write for a given API endpoint (missing fields, wrong types, out-of-range values, malformed body, oversized input, unexpected fields) — a very common practical exercise.
- Understand why a 500 response to invalid input is itself a defect, distinct from the "test passing/failing" — this distinction shows real API testing maturity.
- Be able to explain mass-assignment vulnerabilities with a concrete example (a client sneaking in an unauthorized field) — a specific, memorable example that connects functional negative testing to security awareness.

## References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [ISTQB — Boundary Value Analysis](https://glossary.istqb.org/en_US/term/boundary-value-analysis)