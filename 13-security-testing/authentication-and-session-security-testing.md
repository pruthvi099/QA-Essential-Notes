# Authentication & Session Security Testing

## What It Is

This note covers security-specific testing for authentication and session management — beyond the functional login/logout testing covered in [API Authentication](../04-api-testing/api-authentication.md) and [JWT Testing](../04-api-testing/jwt-testing.md) — focusing specifically on session fixation, weak password policy enforcement, and resilience against credential stuffing attacks.

## Why It Matters

- Broken Authentication has consistently been one of the OWASP Top 10's most impactful categories — authentication is the gateway to everything else in an application, and a weakness here can undermine every other security control downstream.
- These are distinct failure modes from what functional auth testing (does login succeed with valid credentials, does it fail with invalid ones) typically checks — session fixation and credential stuffing resilience specifically require thinking like an attacker attempting to *abuse* the authentication flow, not just verify its happy path.
- This extends [API Authentication](../04-api-testing/api-authentication.md) with the specific, additional security lens interviewers expect an SDET with security awareness to bring beyond pure functional correctness.

## How It Works

**Session fixation** — an attack where an attacker sets or knows a victim's session ID *before* they log in, then hijacks the now-authenticated session once the victim logs in with that same ID. The correct defense is regenerating the session ID upon successful authentication — testing for this means verifying the session identifier actually changes after login, not just that login succeeds.

**Weak password policy testing** — verifying the application actually enforces its stated password requirements (minimum length, complexity) server-side, not just via client-side JavaScript validation that can be trivially bypassed by calling the API directly.

**Credential stuffing resilience** — credential stuffing is an attack using large lists of previously breached username/password pairs against a login endpoint, hoping some users reused credentials. Testing resilience means verifying the application has *some* form of rate limiting or account lockout after repeated failed attempts (see [Idempotency & Rate Limiting](../04-api-testing/idempotency-and-rate-limiting.md) for the general rate-limiting testing pattern, applied here specifically to login attempts).

## Example

**Testing session ID regeneration after login — the core session fixation defense:**
```python
def test_session_id_changes_after_login(api_context):
    # Get a session ID BEFORE logging in (e.g., from an initial page visit)
    pre_login_response = api_context.get("/")
    pre_login_session = pre_login_response.headers.get("set-cookie")

    # Log in
    api_context.post("/api/login", data={
        "email": "test@example.com", "password": "Pass@123"
    })

    post_login_response = api_context.get("/api/account")
    post_login_session = post_login_response.headers.get("set-cookie")

    # The session identifier MUST be different after authentication —
    # if it's the SAME, the application is vulnerable to session fixation
    assert pre_login_session != post_login_session
```

**Testing password policy enforcement server-side, bypassing any client-side JS validation entirely:**
```python
import pytest

@pytest.mark.parametrize("weak_password", [
    "short",           # below minimum length
    "alllowercase",    # no complexity
    "12345678",        # numeric only
    "password",        # common/dictionary password
])
def test_password_policy_enforced_server_side(api_context, weak_password):
    # Calling the API DIRECTLY, bypassing any client-side validation —
    # this is exactly what an attacker or a malicious script would do
    response = api_context.post("/api/register", data={
        "email": "newuser@example.com",
        "password": weak_password,
    })

    # Must be rejected server-side regardless of what the UI enforces
    assert response.status == 400
```

**Testing basic credential stuffing resilience — verifying SOME protection kicks in after repeated failed attempts:**
```python
def test_repeated_failed_logins_trigger_protection(api_context):
    for attempt in range(10):
        response = api_context.post("/api/login", data={
            "email": "test@example.com",
            "password": f"wrong-password-{attempt}",
        })

    # After enough failed attempts, the application should respond
    # differently — either rate-limited (429) or account temporarily
    # locked — NOT just silently allow unlimited guessing attempts
    final_attempt = api_context.post("/api/login", data={
        "email": "test@example.com",
        "password": "another-wrong-password",
    })
    assert final_attempt.status in [429, 423]  # 423 = Locked
```

## Production Considerations

- Session regeneration on login should be tested as a standard part of any authentication flow's test coverage, not treated as an edge case — it's a foundational, well-established defense with a simple, direct test.
- Always verify password policy enforcement by calling the API directly rather than only testing through the UI — client-side-only validation is trivially bypassed and provides no real security, only UX convenience.
- Rate limiting/lockout thresholds need to balance security against legitimate user experience (a threshold too aggressive locks out real users who simply mistyped their password a few times) — this is a genuine product decision worth understanding, not just a pure security maximum.

## Common Pitfalls

- Testing password policy only through the UI form, missing that the actual enforcement (or lack thereof) happens — or doesn't happen — server-side, where it actually matters for security.
- Not testing session ID regeneration after login at all, missing a foundational session fixation vulnerability that's simple to test for once known.
- Assuming any rate limiting exists without actually testing it — an unprotected login endpoint under credential stuffing is a genuine, common real-world attack vector, not a hypothetical one.
- Conflating "the login form has a password strength meter" (a UX feature) with "the server actually rejects weak passwords" (the real security control) — these are often, incorrectly, assumed to be the same thing.

## Interview Notes

- Be ready to explain session fixation precisely — the attack mechanism and the session-regeneration defense — and how you'd test for it specifically.
- Understand why client-side-only validation provides no real security, and be able to describe testing password policy enforcement by bypassing the UI entirely.
- Be able to describe what credential stuffing is and a basic, practical test for verifying an application has some defense against it.

## References

- [OWASP — Broken Authentication](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP — Credential Stuffing Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)