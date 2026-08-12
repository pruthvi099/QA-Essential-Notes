# 02 — Automation: Python & TypeScript Playwright

Core Playwright automation concepts, each note covering Python and TypeScript side by side. Read [00-start-here](../00-start-here/) and [01-manual-testing](../01-manual-testing/) first — this folder assumes familiarity with test design and manual testing fundamentals.

## Notes

1. [Playwright Setup](./playwright-setup.md) — Installing Playwright and running your first test in both languages
2. [Locators](./locators.md) — Role/label/text/test-id locator strategies and why they beat CSS/XPath
3. [Auto-Waiting](./auto-waiting.md) — Actionability checks Playwright performs before every action
4. [Assertions](./assertions.md) — Web-first, retrying assertions vs. plain language assertions
5. [Page Object Model](./page-object-model.md) — Structuring tests for maintainability
6. [Fixtures](./fixtures.md) — Pytest fixtures vs. Playwright Test fixtures, scoping and composition
7. [Hooks & Configuration](./hooks-and-config.md) — `conftest.py`/`pytest.ini` vs. `playwright.config.ts`
8. [Browser Contexts & Test Isolation](./browser-contexts-and-test-isolation.md) — How Playwright isolates tests by default
9. [Handling Authentication](./handling-authentication.md) — Reusing storage state to skip UI login on every test
10. [Parallel Execution](./parallel-execution.md) — `pytest-xdist` vs. Playwright Test's native parallelism
11. [Retries & Timeouts](./retries-and-timeouts.md) — Timeout hierarchy and test-level retry strategy
12. [Tracing, Screenshots & Videos](./tracing-screenshots-videos.md) — Debugging CI failures with the trace viewer
13. [API Testing with Playwright](./api-testing-with-playwright.md) — Using `APIRequestContext` to combine API and UI testing
14. [Handling Dynamic Elements](./handling-dynamic-elements.md) — iframes, shadow DOM, and dynamically loaded content
15. [File Upload & Download](./file-upload-download.md) — Handling file inputs and browser downloads
16. [Flaky Test Handling](./flaky-test-handling.md) — Diagnosing and fixing flaky tests at the root cause

## Related

- [01 — Manual Testing](../01-manual-testing/) — the manual test design skills this folder automates
- [04 — API Testing](../04-api-testing/) — deeper API testing concepts beyond Playwright's built-in client
- [10 — Test Framework Design](../10-test-framework-design/) — where these building blocks combine into a full framework