# JWT Testing

## What It Is

A JSON Web Token (JWT) is a compact, self-contained token format used for stateless authentication — the token itself carries claims (user ID, roles, expiry) rather than requiring the server to look up session state. A JWT has three base64url-encoded parts separated by dots: `header.payload.signature`. Testing JWTs means verifying both the token's structure/claims and how the API behaves with valid, expired, malformed, and tampered tokens.

## Why It Matters

- JWT-based auth is extremely common in modern APIs, and it's stateless by design — meaning the server trusts the token's claims without a database lookup, which makes signature verification and expiry handling critical, high-stakes things to test correctly.
- A JWT implementation bug (accepting an expired token, not verifying the signature, trusting claims that should be re-checked server-side) is a serious security vulnerability, not just a functional bug — this makes JWT testing a place where thoroughness has outsized real-world consequences.
- This is a very commonly asked SDET/API testing interview topic specifically because JWT structure and claims are concrete and testable in a way that's easy to demonstrate hands-on knowledge of, versus vaguer auth concepts.

## How It Works

**JWT structure:** `header.payload.signature`
- **Header** — algorithm and token type (e.g., `{"alg": "HS256", "typ": "JWT"}`)
- **Payload** — claims: `sub` (subject/user ID), `exp` (expiry timestamp), `iat` (issued-at), plus custom claims (roles, permissions)
- **Signature** — verifies the token hasn't been tampered with; computed using the header + payload + a secret/private key

**Critical property:** the payload is base64url-*encoded*, not encrypted — anyone can decode and read it (don't put sensitive data in JWT claims). Only the signature actually secures the token against tampering.

**What to test:**
1. Valid token → request succeeds
2. Expired token (`exp` in the past) → request rejected (401)
3. Malformed token (invalid structure) → request rejected, not a server error
4. Tampered token (payload modified, signature no longer matches) → request rejected
5. Token with modified claims but a valid signature for the *original* claims should never occur — if it does, that's the tampering case above
6. Missing token → request rejected (401), distinct from an invalid one if the API differentiates

## Example

**Python — decoding and inspecting a JWT's claims (without verifying signature, for test inspection):**
```python
import jwt  # PyJWT library
import time

def test_jwt_contains_expected_claims(api_context):
    login_response = api_context.post("/api/login", data={
        "email": "test@example.com", "password": "Pass@123"
    })
    token = login_response.json()["access_token"]

    # Decode WITHOUT verification, purely to inspect claims for test assertions
    decoded = jwt.decode(token, options={"verify_signature": False})

    assert decoded["sub"] is not None
    assert "exp" in decoded
    assert decoded["exp"] > time.time()   # not already expired
    assert "roles" in decoded

def test_expired_jwt_is_rejected(api_context):
    # A pre-generated token with exp set in the past (test fixture)
    expired_token = "eyJhbGciOiJIUzI1NiIs...expired_payload...signature"

    response = api_context.get(
        "/api/account",
        headers={"Authorization": f"Bearer {expired_token}"}
    )
    assert response.status == 401

def test_tampered_jwt_is_rejected(api_context):
    login_response = api_context.post("/api/login", data={
        "email": "test@example.com", "password": "Pass@123"
    })
    token = login_response.json()["access_token"]

    # Tamper: change the last character of the signature
    tampered_token = token[:-1] + ("A" if token[-1] != "A" else "B")

    response = api_context.get(
        "/api/account",
        headers={"Authorization": f"Bearer {tampered_token}"}
    )
    assert response.status == 401
```

**TypeScript — equivalent claim inspection and negative tests:**
```typescript
import { test, expect } from '@playwright/test';
import { jwtDecode } from 'jwt-decode';

test('JWT contains expected claims', async ({ request }) => {
  const loginResponse = await request.post('/api/login', {
    data: { email: 'test@example.com', password: 'Pass@123' },
  });
  const { access_token } = await loginResponse.json();

  const decoded: any = jwtDecode(access_token);

  expect(decoded.sub).toBeDefined();
  expect(decoded.exp).toBeGreaterThan(Date.now() / 1000);
  expect(decoded.roles).toBeDefined();
});

test('tampered JWT is rejected', async ({ request }) => {
  const loginResponse = await request.post('/api/login', {
    data: { email: 'test@example.com', password: 'Pass@123' },
  });
  const { access_token } = await loginResponse.json();

  const tampered = access_token.slice(0, -1) + (access_token.endsWith('A') ? 'B' : 'A');

  const response = await request.get('/api/account', {
    headers: { Authorization: `Bearer ${tampered}` },
  });
  expect(response.status()).toBe(401);
});
```

## Production Considerations

- Never assert on a JWT's claims as a substitute for testing actual server-side authorization — a JWT might correctly claim `role: admin`, but the real test is whether the server actually enforces that role correctly on protected endpoints (see the 401 vs. 403 distinction in [HTTP Methods & Status Codes](./http-methods-and-status-codes.md)).
- Decoding a JWT without verifying its signature (as done above, purely for test inspection of claims from a token *you* just legitimately received) is fine for test assertions — but never build test logic that trusts an *externally supplied* token's claims without signature verification, since that's the exact vulnerability class this note is meant to help catch.
- Test token refresh flows explicitly if the API supports them (short-lived access token + longer-lived refresh token) — refresh logic is a common source of subtle bugs (e.g., a refresh token that doesn't actually get invalidated after use).

## Common Pitfalls

- Assuming JWT payload contents are secure/private because they're "encoded" — base64url encoding is trivially reversible, and sensitive data should never be placed in JWT claims.
- Only testing the valid-token happy path and skipping expired/malformed/tampered token scenarios — these are exactly the cases most likely to reveal a real security gap in the API's JWT handling.
- Confusing claim inspection (decoding a token's payload for test assertions) with signature verification (confirming the token wasn't tampered with) — these are different operations serving different testing purposes.
- Not testing that a token's claims are actually *enforced* server-side — a token claiming `role: admin` proves nothing about security if the server doesn't actually check that claim on protected endpoints.

## Interview Notes

- Be ready to describe a JWT's three parts and explain, precisely, that the payload is encoded (not encrypted) — a very commonly asked, easy-to-get-right-or-wrong fundamental.
- Be able to list the categories of JWT tests you'd write (valid, expired, malformed, tampered, missing) — a common practical "how would you test JWT auth" interview question.
- Understand why claim inspection alone doesn't verify security — the server must actually *enforce* claims, and testing that enforcement (not just decoding the token) is the real point.

## References

- [JWT.io — Introduction](https://jwt.io/introduction)
- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [OWASP — JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)