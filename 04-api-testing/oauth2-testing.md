# OAuth2 Testing

## What It Is

OAuth2 is an authorization framework that lets a client access a resource on a user's behalf without ever handling the user's actual credentials — instead, the client obtains an access token through one of several defined "flows." Testing OAuth2 means verifying the correct flow behaves as specified, tokens are scoped and expire correctly, and the API properly enforces the permissions (scopes) a token was actually granted.

## Why It Matters

- OAuth2 is the standard for third-party integrations (e.g., "Sign in with Google," an app accessing a user's calendar) — testing it correctly matters because a flawed implementation can grant broader access than intended, a serious security issue, not just a functional bug.
- OAuth2 has several distinct flows for different use cases (user-facing login vs. service-to-service), and using or testing the wrong flow for a given scenario reflects a real misunderstanding — interviewers use this to distinguish candidates who've only used OAuth2 as a consumer from those who understand it well enough to test it properly.
- This builds on [JWT Testing](./jwt-testing.md) — OAuth2 access tokens are very often implemented as JWTs, but OAuth2 itself is the authorization *framework* around obtaining and scoping that token, a distinct layer worth testing separately.

## How It Works

**Common OAuth2 flows:**

| Flow | Use Case | Involves a user? |
|---|---|---|
| Authorization Code | Web/mobile apps where a user logs in and grants consent | Yes |
| Client Credentials | Service-to-service (machine-to-machine) auth, no user involved | No |
| Refresh Token | Obtaining a new access token without re-authenticating | No (uses a prior grant) |

**Authorization Code flow (simplified):**
1. Client redirects the user to the authorization server's login/consent screen
2. User logs in and grants permission (consents to requested scopes)
3. Authorization server redirects back with a short-lived authorization code
4. Client exchanges that code (server-side, with a client secret) for an access token

**Client Credentials flow (simplified, most common in pure API testing):**
1. Client sends its `client_id` and `client_secret` directly to the token endpoint
2. Authorization server returns an access token — no user or redirect involved

**Scopes** define what an access token is actually permitted to do (`read:orders`, `write:orders`) — a valid, unexpired token should still be rejected for an action outside its granted scopes.

## Example

**Python — Client Credentials flow (the most common flow to test directly in an API suite):**
```python
def test_client_credentials_flow(api_context):
    token_response = api_context.post("/oauth/token", data={
        "grant_type": "client_credentials",
        "client_id": "test-client-id",
        "client_secret": "test-client-secret",
        "scope": "read:orders",
    })
    assert token_response.status == 200
    token_data = token_response.json()
    assert "access_token" in token_data
    assert token_data["token_type"] == "Bearer"

    # Use the obtained token against a protected endpoint
    access_token = token_data["access_token"]
    orders_response = api_context.get(
        "/api/orders",
        headers={"Authorization": f"Bearer {access_token}"}
    )
    assert orders_response.status == 200

def test_token_rejected_for_out_of_scope_action(api_context):
    token_response = api_context.post("/oauth/token", data={
        "grant_type": "client_credentials",
        "client_id": "test-client-id",
        "client_secret": "test-client-secret",
        "scope": "read:orders",   # read-only scope
    })
    access_token = token_response.json()["access_token"]

    # Attempting a WRITE action with a read-only-scoped token
    write_response = api_context.post(
        "/api/orders",
        headers={"Authorization": f"Bearer {access_token}"},
        data={"items": [{"product_id": 101, "qty": 1}]}
    )
    # Correctly authenticated (valid token) but not authorized (wrong scope)
    assert write_response.status == 403

def test_invalid_client_credentials_rejected(api_context):
    response = api_context.post("/oauth/token", data={
        "grant_type": "client_credentials",
        "client_id": "test-client-id",
        "client_secret": "wrong-secret",
        "scope": "read:orders",
    })
    assert response.status == 401
```

**TypeScript — refresh token flow:**
```typescript
import { test, expect } from '@playwright/test';

test('refresh token issues a new valid access token', async ({ request }) => {
  const initialToken = await request.post('/oauth/token', {
    data: {
      grant_type: 'client_credentials',
      client_id: 'test-client-id',
      client_secret: 'test-client-secret',
    },
  });
  const { refresh_token } = await initialToken.json();

  const refreshResponse = await request.post('/oauth/token', {
    data: {
      grant_type: 'refresh_token',
      refresh_token,
      client_id: 'test-client-id',
      client_secret: 'test-client-secret',
    },
  });

  expect(refreshResponse.status()).toBe(200);
  const { access_token: newAccessToken } = await refreshResponse.json();
  expect(newAccessToken).toBeDefined();

  // The new token should actually work against a protected endpoint
  const ordersResponse = await request.get('/api/orders', {
    headers: { Authorization: `Bearer ${newAccessToken}` },
  });
  expect(ordersResponse.status()).toBe(200);
});
```

## Production Considerations

- Client Credentials flow is by far the most practical to test directly in an automated API suite — Authorization Code flow involves a real user login/consent redirect, which typically needs to be tested via UI automation (or a specialized test-mode bypass some auth providers offer) rather than pure API calls.
- Scope enforcement testing (verifying a token can't perform actions outside its granted scope) is one of the highest-value OAuth2 tests — this is where real authorization bugs (over-privileged tokens) tend to live, distinct from simple "is the token valid" checks.
- Store client secrets used in test suites the same way as any other credential — environment variables and CI secret storage, never hardcoded or committed (same principle as [API Authentication](./api-authentication.md)).

## Common Pitfalls

- Testing only that a token was successfully obtained, without testing that it's correctly scope-restricted — this misses the authorization enforcement bugs that matter most from a security standpoint.
- Confusing "invalid token" (401 — not authenticated) with "valid token, wrong scope" (403 — authenticated but not authorized) — the same 401/403 distinction from [HTTP Methods & Status Codes](./http-methods-and-status-codes.md) applies directly here.
- Attempting to fully automate the Authorization Code flow's user-consent redirect through pure API calls without understanding it actually requires a browser/user interaction step (or a provider-specific test bypass) — this is a common point of confusion for testers newer to OAuth2.
- Not testing refresh token behavior (does the old refresh token get invalidated after use? does an expired refresh token correctly fail?) — refresh flow bugs are a common, real source of session-related security issues.

## Interview Notes

- Be ready to explain the difference between Authorization Code flow and Client Credentials flow, and when each is used — a very common OAuth2 fundamentals question.
- Understand scopes and be able to describe a test verifying a token is correctly *rejected* for an out-of-scope action — this is the practical core of OAuth2 authorization testing.
- Be able to explain why Authorization Code flow is harder to test purely at the API level than Client Credentials flow, due to the user consent step.

## References

- [OAuth 2.0 — Official Site](https://oauth.net/2/)
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)