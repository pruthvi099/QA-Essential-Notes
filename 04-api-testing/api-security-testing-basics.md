# API Security Testing Basics

## What It Is

API security testing covers the common vulnerability classes an SDET should recognize and test for at the API layer — beyond functional correctness, checking whether an API properly enforces access control, avoids leaking excess data, and resists common injection/abuse patterns. This isn't a substitute for dedicated penetration testing, but a practical baseline every API test suite should include.

## Why It Matters

- Security bugs at the API layer are often more severe than functional bugs — a broken button is an inconvenience; a broken authorization check can expose every user's data.
- Many API security issues are straightforward to test for once you know the category — this makes API security testing a high-leverage skill: relatively simple tests, catching high-severity issues.
- This connects several earlier notes (negative testing, JWT/OAuth2, request validation) into a security-specific lens — interviewers use API security questions to see whether a candidate thinks beyond "does the feature work" toward "can this be misused."

## How It Works

**Common API vulnerability categories an SDET can test for directly (based on the OWASP API Security Top 10):**

1. **Broken Object Level Authorization (BOLA)** — can a user access another user's resource by simply changing an ID in the request? (e.g., `GET /api/orders/501` belonging to another user)
2. **Broken Authentication** — weak token handling, missing expiry, predictable tokens (see [JWT Testing](./jwt-testing.md))
3. **Excessive Data Exposure** — does a response include more fields than the client actually needs, including sensitive internal fields?
4. **Mass Assignment** — can a client set fields it shouldn't control by including them in the request body? (see [Negative Testing for APIs](./negative-testing-for-apis.md))
5. **Injection** — does the API properly sanitize input used in database queries, commands, or other interpreters?
6. **Lack of Rate Limiting** — can an endpoint be abused via unlimited requests? (see [Idempotency & Rate Limiting](./idempotency-and-rate-limiting.md))
7. **Security Misconfiguration** — verbose error messages leaking stack traces/internal details, missing security headers, overly permissive CORS settings

## Example

**Python — testing BOLA (one of the most common, highest-impact API vulnerabilities):**
```python
def test_user_cannot_access_another_users_order(api_context):
    # User A creates an order
    user_a_token = login_and_get_token(api_context, "user_a@example.com")
    create_response = api_context.post(
        "/api/orders",
        headers={"Authorization": f"Bearer {user_a_token}"},
        data={"items": [{"product_id": 101, "qty": 1}]},
    )
    order_id = create_response.json()["id"]

    # User B attempts to access User A's order directly by ID
    user_b_token = login_and_get_token(api_context, "user_b@example.com")
    response = api_context.get(
        f"/api/orders/{order_id}",
        headers={"Authorization": f"Bearer {user_b_token}"},
    )

    # This MUST be rejected — if it returns 200, that's a critical
    # BOLA vulnerability exposing another user's private data
    assert response.status in [403, 404]
```

**Python — testing for excessive data exposure:**
```python
def test_user_profile_does_not_expose_sensitive_fields(api_context, user_token):
    response = api_context.get(
        "/api/users/me",
        headers={"Authorization": f"Bearer {user_token}"}
    )
    body = response.json()

    # Fields that should NEVER appear in a client-facing response
    sensitive_fields = ["password_hash", "internal_notes", "ssn", "credit_card_full"]
    for field in sensitive_fields:
        assert field not in body, f"Sensitive field '{field}' exposed in API response"
```

**Python — a basic injection probe (not a substitute for dedicated security tooling, but a useful baseline check):**
```python
def test_search_endpoint_handles_sql_injection_attempt_safely(api_context):
    malicious_input = "'; DROP TABLE orders; --"
    response = api_context.get(f"/api/products/search?q={malicious_input}")

    # Should handle it as ordinary (likely empty-result) input,
    # not error out or expose any indication the input reached a raw query
    assert response.status in [200, 400]
    assert "DROP TABLE" not in response.text()
```

**TypeScript — testing security headers/misconfiguration:**
```typescript
import { test, expect } from '@playwright/test';

test('API error responses do not leak stack traces', async ({ request }) => {
  const response = await request.get('/api/orders/not-a-valid-id');

  expect(response.status()).toBe(400);
  const body = await response.text();

  // A verbose stack trace in a client-facing error response is a
  // real information-leakage risk, revealing internal implementation details
  expect(body.toLowerCase()).not.toContain('stacktrace');
  expect(body.toLowerCase()).not.toContain('at com.example.internal');
});
```

## Production Considerations

- BOLA testing (accessing another user's resource by ID manipulation) should be a standard, repeated check across *every* resource-owning endpoint in an API, not a one-off test — it's consistently one of the most common and severe real-world API vulnerabilities (OWASP API Security's #1 category).
- These tests are a baseline, not a replacement for dedicated security testing/penetration testing by specialists — an SDET catching these categories in everyday API testing adds real value, but doesn't substitute for a proper security audit on high-risk systems.
- Report any confirmed security finding (a genuine BOLA, exposed sensitive field, working injection) through your organization's security/responsible-disclosure process, not just as a regular bug ticket — these often need different handling, urgency, and confidentiality than typical functional defects.

## Common Pitfalls

- Only testing that a user CAN access their own resources, never testing that they CANNOT access others' — this misses BOLA entirely, since the happy path alone gives no signal about authorization boundaries.
- Assuming HTTPS alone makes an API "secure" — transport encryption protects data in transit, but does nothing for authorization bugs, data exposure, or injection vulnerabilities at the application layer.
- Treating a verbose error message (stack trace, internal file paths) as merely a "messy" response instead of a real information-leakage security issue worth flagging.
- Skipping basic security checks entirely because "that's the security team's job" — foundational checks like BOLA and data exposure are well within an SDET's everyday scope and are cheapest to catch early, in the same test suite as functional coverage.

## Interview Notes

- Be ready to explain BOLA with a concrete example (changing an ID in a URL to access another user's data) — this is one of the most commonly referenced API vulnerabilities in interviews, given its real-world frequency and severity.
- Understand the distinction between authentication testing (JWT/OAuth2 validity) and authorization testing (BOLA — does a *valid* user have the *right* to this specific resource) — conflating the two is a common gap.
- Be able to name a few OWASP API Security Top 10 categories and describe a simple test for at least one — shows practical, not just theoretical, security awareness.

## References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [OWASP — Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)