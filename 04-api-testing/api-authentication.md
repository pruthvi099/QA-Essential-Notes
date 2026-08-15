# API Authentication (API Keys, Basic Auth, Session-Based)

## What It Is

API authentication verifies the identity of the client making a request. Several mechanisms exist, each with different testing implications: **API keys** (a static secret sent per request), **Basic Auth** (username/password base64-encoded in a header), and **session-based auth** (a server-issued session cookie/token after login). This note covers these three; token-based auth (JWT) and OAuth2 are deep enough to warrant their own notes (see [JWT Testing](./jwt-testing.md) and [OAuth2 Testing](./oauth2-testing.md)).

## Why It Matters

- Nearly every real API test suite needs to handle authentication correctly before testing anything else — misconfigured or misunderstood auth setup is a common source of tests that fail for the wrong reason (auth issues masquerading as feature bugs).
- Different auth mechanisms have different testing needs: API keys need rotation/expiry testing, Basic Auth needs encoding verification, session-based auth needs cookie/session lifecycle testing — treating them identically misses mechanism-specific risks.
- This directly extends [Handling Authentication](../02-automation-python-playwright/handling-authentication.md), which covered UI-driven session reuse — this note focuses on authenticating purely at the API level, which is faster and more common in dedicated API test suites.

## How It Works

**API Keys** — a static string sent in a header (`X-API-Key`) or query parameter, identifying the calling application/client (not necessarily a specific user). Simple, but the key itself is a long-lived secret that needs careful handling.

**Basic Auth** — `username:password` base64-encoded and sent in the `Authorization: Basic <encoded>` header. Simple but sends credentials on every request (must be used over HTTPS only) and has no built-in expiry — it's largely legacy for user-facing APIs but still common for internal/service-to-service auth.

**Session-Based Auth** — client logs in via a dedicated endpoint, server returns a session identifier (often as a cookie), and the client includes it on subsequent requests. The server tracks session state; this is stateful, unlike the stateless mechanisms above.

## Example

**Python — API key auth:**
```python
import pytest

@pytest.fixture
def api_context(playwright):
    context = playwright.request.new_context(
        base_url="https://api.example.com",
        extra_http_headers={"X-API-Key": "test-api-key-abc123"},
    )
    yield context
    context.dispose()

def test_api_key_required(playwright):
    # No API key provided — should be rejected
    context = playwright.request.new_context(base_url="https://api.example.com")
    response = context.get("/api/products")
    assert response.status == 401
    context.dispose()

def test_api_key_valid(api_context):
    response = api_context.get("/api/products")
    assert response.status == 200
```

**Python — Basic Auth:**
```python
import base64

def test_basic_auth_valid_credentials(playwright):
    credentials = base64.b64encode(b"test_user:Pass@123").decode("utf-8")
    context = playwright.request.new_context(
        base_url="https://api.example.com",
        extra_http_headers={"Authorization": f"Basic {credentials}"},
    )
    response = context.get("/api/account")
    assert response.status == 200
    context.dispose()

def test_basic_auth_invalid_credentials(playwright):
    credentials = base64.b64encode(b"test_user:WrongPassword").decode("utf-8")
    context = playwright.request.new_context(
        base_url="https://api.example.com",
        extra_http_headers={"Authorization": f"Basic {credentials}"},
    )
    response = context.get("/api/account")
    assert response.status == 401
    context.dispose()
```

**TypeScript — session-based auth (login, then reuse the session cookie):**
```typescript
import { test, expect, request } from '@playwright/test';

test('session-based auth flow', async () => {
  const context = await request.newContext({ baseURL: 'https://api.example.com' });

  // Login sets a session cookie automatically on this context
  const loginResponse = await context.post('/api/login', {
    data: { email: 'test@example.com', password: 'Pass@123' },
  });
  expect(loginResponse.status()).toBe(200);

  // Subsequent requests on the SAME context automatically include
  // the session cookie — no manual header management needed
  const accountResponse = await context.get('/api/account');
  expect(accountResponse.status()).toBe(200);

  // Logout should invalidate the session
  await context.post('/api/logout');
  const afterLogoutResponse = await context.get('/api/account');
  expect(afterLogoutResponse.status()).toBe(401);

  await context.dispose();
});
```

## Production Considerations

- Never hardcode real API keys or credentials directly in test code or version control — use environment variables and CI secret storage, treating test credentials with the same care as production ones, since a leaked test API key can still grant real access.
- For session-based auth, explicitly test session expiry and logout invalidation — these are common, real sources of production security bugs (a "logged out" session that still works is a genuine defect, not just a test edge case).
- Prefer a dedicated test/service account over reusing a real user's credentials for automated API auth — this keeps test activity isolated and auditable, and avoids disrupting a real account's state.

## Common Pitfalls

- Testing only the "valid credentials" happy path and skipping invalid/missing credential scenarios — auth is exactly the kind of area where negative testing (see [Negative Testing for APIs](./negative-testing-for-apis.md)) matters most, since auth failures have security implications.
- Not testing session/token expiry at all, assuming a session lasts forever — this misses a whole class of real bugs around session lifecycle management.
- Sending Basic Auth credentials over plain HTTP in a test environment "since it's just testing" — this normalizes an insecure pattern that could leak into how the API is actually deployed or documented.
- Confusing API key auth (identifies the calling application) with session/user auth (identifies a specific logged-in user) — these serve different purposes and are often used together, not interchangeably.

## Interview Notes

- Be ready to compare API keys, Basic Auth, and session-based auth — their trade-offs, and what each is typically used for (service-to-service vs. user-facing).
- Understand why Basic Auth must be used only over HTTPS, and be able to explain the security risk of sending it over plain HTTP.
- Be able to describe how you'd test session expiry and logout invalidation specifically — a common, practical follow-up question distinguishing thorough auth testing from surface-level coverage.

## References

- [MDN — HTTP Authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)