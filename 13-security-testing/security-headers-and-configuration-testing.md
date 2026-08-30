# Security Headers & Configuration Testing

## What It Is

This note covers testing HTTP security headers — Content Security Policy (CSP), HTTP Strict Transport Security (HSTS), `X-Content-Type-Options`, and the clickjacking-related headers introduced in [CSRF & Clickjacking Testing](./csrf-and-clickjacking-testing.md) — as well as broader security misconfiguration checks (verbose error messages, exposed debug endpoints, default credentials). This is one of the most accessible, checklist-friendly categories of security testing an SDET can own directly.

## Why It Matters

- Security Misconfiguration is a persistent OWASP Top 10 category precisely because these issues are common, easy to introduce accidentally (a debug flag left on, a header simply never configured), and — critically — easy to check for, making this some of the highest-value, lowest-effort security testing available to a functional SDET.
- Missing or misconfigured security headers don't require any special exploitation skill to test for — checking their presence and correct values is straightforward, repeatable, and highly automatable, making it an ideal candidate for a standard CI check (see [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md)).
- This ties together several defenses referenced elsewhere in this folder (CSP as XSS defense-in-depth, clickjacking headers) into one systematic verification pass.

## How It Works

**Key security headers to verify:**

| Header | Purpose |
|---|---|
| `Content-Security-Policy` | Restricts what sources scripts/styles/resources can load from — a defense-in-depth layer against XSS (see [XSS Testing](./xss-testing.md)) |
| `Strict-Transport-Security` (HSTS) | Forces browsers to only connect via HTTPS, preventing downgrade/man-in-the-middle attacks on subsequent visits |
| `X-Content-Type-Options: nosniff` | Prevents the browser from MIME-sniffing a response away from its declared Content-Type, reducing certain content-type-confusion attacks |
| `X-Frame-Options` / CSP `frame-ancestors` | Clickjacking protection (see [CSRF & Clickjacking Testing](./csrf-and-clickjacking-testing.md)) |
| `Set-Cookie` attributes (`Secure`, `HttpOnly`, `SameSite`) | Ensures cookies are only sent over HTTPS, inaccessible to JavaScript (mitigating XSS-based cookie theft), and restricted in cross-site contexts |

**Broader security misconfiguration checks:**
- Verbose error messages/stack traces exposed to end users (connects to the information-leakage point from [Injection Attacks & Testing](./injection-attacks-and-testing.md)).
- Debug/admin endpoints accidentally exposed in production.
- Default or overly permissive CORS configuration.
- Directory listing enabled on a web server, exposing file structure.

## Example

**A systematic header-check test, verifying multiple security headers in one pass:**
```python
def test_security_headers_present_and_correct(page):
    response = page.goto("/")
    headers = {k.lower(): v for k, v in response.headers.items()}

    assert "content-security-policy" in headers, "Missing CSP header"

    assert "strict-transport-security" in headers, "Missing HSTS header"
    assert "max-age=" in headers.get("strict-transport-security", "")

    assert headers.get("x-content-type-options") == "nosniff"

    has_clickjack_protection = (
        "x-frame-options" in headers or "frame-ancestors" in headers.get("content-security-policy", "")
    )
    assert has_clickjack_protection

def test_session_cookie_has_secure_attributes(page, api_context):
    api_context.post("/api/login", data={"email": "test@example.com", "password": "Pass@123"})

    set_cookie_header = api_context.get("/api/account").headers.get("set-cookie", "")

    assert "Secure" in set_cookie_header, "Session cookie missing Secure attribute"
    assert "HttpOnly" in set_cookie_header, "Session cookie missing HttpOnly attribute (XSS can steal it otherwise)"
    assert "SameSite" in set_cookie_header, "Session cookie missing SameSite attribute (CSRF exposure)"
```

**Testing for verbose error exposure — extending the same information-leakage pattern from earlier notes:**
```python
def test_404_page_does_not_expose_stack_trace(page):
    response = page.goto("/this-route-does-not-exist")
    content = page.content().lower()

    assert response.status == 404
    assert "traceback" not in content
    assert "stack trace" not in content
    assert "at com.example" not in content   # framework-specific internal path patterns
    assert "django.db" not in content         # ORM/framework internals shouldn't leak
```

**Checking for an exposed debug endpoint that should never be reachable in production:**
```python
def test_debug_endpoint_not_accessible_in_production(api_context):
    common_debug_paths = ["/debug", "/__debug__", "/admin/debug", "/.env", "/actuator"]

    for path in common_debug_paths:
        response = api_context.get(path)
        assert response.status in [404, 403], f"Debug path {path} unexpectedly accessible ({response.status})"
```

## Production Considerations

- Header checks are ideal candidates for automated CI gates (see [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md)) — they're fast, deterministic, and catch regressions immediately (a header accidentally removed during an unrelated config change) rather than only during periodic manual review.
- Verify security headers per environment — a header correctly configured in production but missing in staging (or vice versa) can hide a real gap; test against the environment that actually matters for the specific check (e.g., HSTS is meaningless to test over plain HTTP in a local dev environment).
- CSP configuration is often iteratively tightened over a project's life — a very permissive initial CSP (`unsafe-inline` allowed broadly) provides much weaker XSS defense-in-depth than a tightly scoped one; periodically reviewing CSP strictness, not just its presence, is worth doing.

## Common Pitfalls

- Checking only for a header's *presence* without verifying its *value* is actually protective (e.g., a `Content-Security-Policy` header that exists but is set to something so permissive it provides little real protection).
- Testing security headers only on the homepage, missing that different routes/environments can have inconsistent configuration.
- Not testing cookie security attributes (`Secure`, `HttpOnly`, `SameSite`) specifically, treating "the session works" as sufficient without checking these often-overlooked but important attributes.
- Assuming a debug/admin endpoint is safely hidden because it's "not linked from the UI" — obscurity is not a real security control, and these endpoints need to be explicitly tested as inaccessible, not just assumed to be.

## Interview Notes

- Be ready to name several key security headers and explain what each specifically protects against — a common, practical checklist-style interview question.
- Understand why header presence alone isn't sufficient — the actual configured value matters (an overly permissive CSP is a common, specific example worth citing).
- Be able to describe cookie security attributes (`Secure`, `HttpOnly`, `SameSite`) and what each protects against specifically — this is a frequently asked, precise detail.

## References

- [OWASP — Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
- [OWASP — HTTP Security Response Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- [MDN — HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)