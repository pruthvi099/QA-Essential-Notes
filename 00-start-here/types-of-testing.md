
# Types of Testing

## What It Is

Types of testing describe *what aspect* of the software is being validated, as opposed to *levels of testing* (which describe scope — see [Levels of Testing](./levels-of-testing.md)). Types fall into two broad categories:

- **Functional Testing** — does the software do what it's supposed to do? (features, business logic, UI behavior, APIs)
- **Non-Functional Testing** — does the software behave well under real-world conditions? (performance, security, usability, reliability, compatibility)

Any level of testing (unit, integration, system) can be functional or non-functional — the two axes are independent.

## Why It Matters

- Most beginners equate "testing" with functional testing only. Production incidents are frequently caused by non-functional gaps — a feature that works correctly but is too slow, insecure, or breaks on a different browser/device.
- As an SDET, you're expected to know which type of testing is appropriate for a given risk, and to be able to justify test strategy decisions ("we automated functional regression, but performance testing is run manually/quarterly because it needs dedicated tooling").
- Interviewers commonly probe whether you can classify testing correctly, since it signals whether you understand testing as an engineering discipline rather than "clicking buttons."

## How It Works

**Functional testing types:**

| Type | Focus |
|---|---|
| Smoke Testing | Quick check that critical paths work after a build — "is it worth testing further?" |
| Sanity Testing | Narrow, quick check that a specific bug fix/change works, without full regression |
| Regression Testing | Re-running existing tests to confirm nothing broke after a change |
| Integration Testing | Modules/components work together correctly |
| System Testing | Full application meets requirements end-to-end |
| UAT | Business/end users confirm the system meets their needs |

**Non-functional testing types:**

| Type | Focus |
|---|---|
| Performance Testing | Speed, responsiveness, stability under load |
| Security Testing | Vulnerabilities, unauthorized access, data protection |
| Usability Testing | Ease of use, user experience |
| Compatibility Testing | Works across browsers, devices, OS versions |
| Reliability Testing | Behaves consistently over time (e.g., no memory leaks) |
| Accessibility Testing | Usable by people with disabilities (WCAG compliance) |

## Example

Smoke vs. Regression, expressed as pytest markers — a common real-world pattern for controlling *which type* of test runs at each CI stage:

```python
import pytest

@pytest.mark.smoke
def test_homepage_loads(page):
    page.goto("https://example.com")
    assert page.title() != ""

@pytest.mark.smoke
def test_login_works(page):
    page.goto("https://example.com/login")
    page.fill("#email", "user@example.com")
    page.fill("#password", "Pass@123")
    page.click("#login-btn")
    assert page.url.endswith("/dashboard")

@pytest.mark.regression
def test_password_reset_flow(page):
    page.goto("https://example.com/forgot-password")
    page.fill("#email", "user@example.com")
    page.click("#submit-btn")
    assert page.get_by_text("Reset link sent").is_visible()
```

```bash
# CI runs smoke tests on every push (fast feedback)
pytest -m smoke

# Full regression runs nightly or before release (slower, broader)
pytest -m regression
```

## Production Considerations

- Smoke tests should be small (minutes, not tens of minutes) and cover only the paths that, if broken, mean the build isn't worth testing further (login, homepage, checkout, etc.).
- Non-functional testing (performance, security) often requires dedicated tools (JMeter/k6, OWASP ZAP/Burp) and specialist ownership — it's rarely something a functional automation suite covers as a side effect.
- Regression suites grow over time; without pruning/tagging strategy