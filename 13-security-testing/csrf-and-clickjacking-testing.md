# CSRF & Clickjacking Testing

## What It Is

**Cross-Site Request Forgery (CSRF)** tricks an authenticated user's browser into submitting an unwanted request to an application they're logged into, without their knowledge — exploiting the fact that browsers automatically attach cookies to requests regardless of which site initiated them. **Clickjacking** tricks a user into clicking something different from what they perceive, typically by overlaying an invisible, legitimate page (via an iframe) beneath deceptive visible content. Both attacks exploit user *trust and interaction* rather than a direct application flaw, distinguishing them from injection/XSS.

## Why It Matters

- CSRF and clickjacking are both attacks that succeed *because* the victim is legitimately authenticated — this makes them a different risk category from injection/XSS, worth understanding as distinct: the application isn't necessarily flawed in handling malicious input, it's flawed in not verifying that a request or a click genuinely originated from an intended, first-party context.
- Both have well-established, testable defenses (CSRF tokens, `X-Frame-Options`/CSP `frame-ancestors`) — this makes them practical, checkable items rather than abstract risks, well within a functional SDET's testing scope.
- These are commonly paired in security-awareness interview questions specifically because they're often confused with each other or with XSS — being able to distinguish all three precisely is a good, specific signal of real understanding.

## How It Works

**CSRF — the attack:** a victim is logged into `bank.com` in one browser tab. A malicious page (`evil.com`) contains a hidden auto-submitting form targeting `bank.com/transfer`. Because the browser automatically attaches `bank.com`'s session cookie to any request sent to `bank.com`, the malicious request appears legitimately authenticated — even though the victim never intended to submit it.

**CSRF — the defense:** a CSRF token — a random, unpredictable value generated per session (or per request) and required as part of any state-changing request. Since `evil.com` has no way to know or read this token (same-origin policy prevents it from reading `bank.com`'s page content), it can't include a valid token in its forged request, and the server rejects the request.

**Clickjacking — the attack:** a malicious page overlays a legitimate page (e.g., `bank.com/confirm-transfer`) in a transparent iframe beneath deceptive visible content (a "Click to win a prize" button positioned exactly over the real page's "Confirm Transfer" button) — the victim thinks they're clicking the visible content, but they're actually clicking the hidden, legitimate page underneath.

**Clickjacking — the defense:** the `X-Frame-Options` header (or the more modern CSP `frame-ancestors` directive) tells the browser whether the page is allowed to be embedded in an iframe on another site at all — if set to deny/same-origin, the browser refuses to render the page in a malicious iframe, defeating the attack before it can even display.

## Example

**Testing that a state-changing endpoint requires a valid CSRF token:**
```python
def test_state_changing_request_requires_csrf_token(api_context):
    # Attempting a state-changing action WITHOUT a CSRF token,
    # simulating a forged cross-site request
    response = api_context.post(
        "/api/account/change-email",
        data={"new_email": "attacker-controlled@evil.com"},
        headers={},  # deliberately omitting the CSRF token header
    )

    # Should be REJECTED — a properly protected endpoint requires
    # a valid token for any state-changing (non-GET) request
    assert response.status == 403

def test_state_changing_request_succeeds_with_valid_csrf_token(api_context, csrf_token):
    response = api_context.post(
        "/api/account/change-email",
        data={"new_email": "legitimate-new-email@example.com"},
        headers={"X-CSRF-Token": csrf_token},
    )
    assert response.status == 200
```

**Testing that clickjacking protection headers are present:**
```python
def test_response_includes_clickjacking_protection_headers(page):
    response = page.goto("/account/confirm-transfer")
    headers = response.headers

    # Either header provides protection; modern CSP frame-ancestors
    # is generally preferred, but X-Frame-Options remains widely used
    # as a broadly-compatible fallback
    has_xfo = "x-frame-options" in headers
    has_csp_frame_ancestors = (
        "content-security-policy" in headers
        and "frame-ancestors" in headers.get("content-security-policy", "")
    )
    assert has_xfo or has_csp_frame_ancestors, (
        "Sensitive page has NO clickjacking protection header set"
    )
```

**A manual/exploratory verification for clickjacking — actually attempting to frame the page, the kind of check worth doing at least once manually:**
```html
<!-- test-clickjack.html — a minimal local test harness -->
<iframe src="https://staging.example.com/account/confirm-transfer" width="600" height="400"></iframe>
<!-- If X-Frame-Options/CSP is correctly configured, the browser
     refuses to render this iframe's content at all — a blank/
     blocked iframe is the CORRECT, expected result -->
```

## Production Considerations

- CSRF protection specifically needs to apply to every state-changing endpoint (POST, PUT, PATCH, DELETE) — testing just one endpoint and assuming the pattern holds everywhere is a common, risky gap; a systematic sweep across all mutating endpoints is worth doing.
- Clickjacking protection headers should be verified specifically on sensitive, action-triggering pages (payment confirmation, account changes) — these are the highest-value targets for a clickjacking attack, even if the header is technically applied site-wide.
- Modern frameworks often include CSRF protection by default (e.g., Django, Rails) — but verify it's actually enabled and not accidentally disabled for a specific endpoint (a common real-world gap, especially around API endpoints added later that may not have inherited the framework's default protection).

## Common Pitfalls

- Testing CSRF protection on only one endpoint and assuming it's uniformly applied across the entire application, missing a specific mutating endpoint that lacks the protection.
- Confusing CSRF with XSS — CSRF exploits trust in an authenticated *user's browser* making a forged request; XSS exploits trust in *content* being rendered as executable script — these are mechanically distinct attacks with distinct defenses, a very common point of confusion worth being precise about.
- Assuming a framework's default CSRF protection is automatically applied to every endpoint, including newer API endpoints that may have been added outside the framework's standard request-handling path.
- Checking for the presence of `X-Frame-Options`/CSP headers only on the homepage, missing that the header needs to be verified specifically on the actual sensitive, high-value pages an attacker would target.

## Interview Notes

- Be ready to explain CSRF and clickjacking as attacks that exploit user trust/authentication rather than an input-handling flaw — and to distinguish both clearly from XSS, a very commonly confused trio.
- Understand exactly why a CSRF token defeats the attack (the attacker's page can't read/know the token due to same-origin policy) — the mechanism, not just "tokens prevent CSRF."
- Be able to describe a basic, practical test for both CSRF protection (missing-token request should fail) and clickjacking protection (header presence, or an actual iframe-embedding attempt).

## References

- [OWASP — Cross-Site Request Forgery (CSRF)](https://owasp.org/www-community/attacks/csrf)
- [OWASP — Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html)