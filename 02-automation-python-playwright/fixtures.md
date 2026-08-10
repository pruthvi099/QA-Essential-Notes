# Fixtures (Python & TypeScript)

## What It Is

Fixtures provide reusable setup and teardown logic for tests — creating a browser, logging in a user, seeding data — without duplicating that logic in every test function. Both ecosystems have a fixture system, but they work differently under the hood:

- **Pytest (Python)** — fixtures are functions decorated with `@pytest.fixture`, requested by name as test function parameters. `pytest-playwright` provides built-in fixtures (`page`, `browser`, `context`).
- **Playwright Test (TypeScript)** — fixtures are defined via `test.extend()`, and built-in fixtures (`page`, `browser`, `context`) come from `@playwright/test` natively, with a more structured dependency model than pytest's.

## Why It Matters

- Fixtures are what keep test code DRY and focused on business logic — without them, every test repeats the same setup (launching a browser, logging in, seeding data), making tests longer and setup changes require editing every test file.
- Fixture *scope* (function, session, etc.) directly affects test speed and isolation — misusing scope is a common cause of either slow suites (re-doing expensive setup every test) or flaky suites (tests sharing state that should have been isolated).
- SDET interviews frequently probe fixture design specifically, since it's one of the clearest signals of framework architecture thinking versus "just writing scripts."

## How It Works

**Pytest fixture scopes** (`@pytest.fixture(scope=...)`):
- `function` (default) — runs before/after every test function
- `class` — once per test class
- `module` — once per file
- `session` — once per entire test run

**Playwright Test fixture scope** is simpler — fixtures are either `test`-scoped (default, per test) or `worker`-scoped (shared across all tests run by the same parallel worker process, similar in spirit to pytest's `session` scope but tied to workers, not the whole run).

Both support **fixture dependencies** — a fixture can request other fixtures, building up a chain of setup (e.g., an `authenticated_page` fixture that depends on `page`).

## Example

**Python (pytest):**
```python
# conftest.py
import pytest
from playwright.sync_api import Page

@pytest.fixture(scope="function")
def authenticated_page(page: Page):
    page.goto("https://example.com/login")
    page.get_by_label("Email").fill("test@example.com")
    page.get_by_label("Password").fill("Pass@123")
    page.get_by_role("button", name="Log in").click()
    page.wait_for_url("**/dashboard")
    yield page   # test runs here
    # teardown (optional) runs after the test
```

```python
# test_dashboard.py
def test_dashboard_shows_username(authenticated_page):
    assert authenticated_page.get_by_test_id("username").inner_text() == "test@example.com"
```

**TypeScript (Playwright Test):**
```typescript
// fixtures.ts
import { test as base, Page } from '@playwright/test';

type MyFixtures = {
  authenticatedPage: Page;
};

export const test = base.extend<MyFixtures>({
  authenticatedPage: async ({ page }, use) => {
    await page.goto('https://example.com/login');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('Pass@123');
    await page.getByRole('button', { name: 'Log in' }).click();
    await page.waitForURL('**/dashboard');
    await use(page);   // test runs here
    // teardown (optional) runs after use()
  },
});

export { expect } from '@playwright/test';
```

```typescript
// dashboard.spec.ts
import { test, expect } from '../fixtures';

test('dashboard shows username', async ({ authenticatedPage }) => {
  await expect(authenticatedPage.getByTestId('username')).toHaveText('test@example.com');
});
```

## Production Considerations

- Prefer `worker` scope (TS) / `session` scope (Python) only for genuinely expensive, safely-shareable setup (e.g., an authenticated storage state saved to disk — see [Handling Authentication](./handling-authentication.md)) — sharing mutable state across tests at this scope is a common source of test interdependence bugs.
- Compose fixtures instead of writing one large fixture that does everything — an `authenticated_page` fixture built on top of a `page` fixture, potentially built on top of an `api_client` fixture for backend seeding, keeps each piece independently reusable.
- In TypeScript, fixtures can be automatically applied to every test via `{ auto: true }` — useful for things like automatic evidence capture on failure (see [Defect Reporting Best Practices](../01-manual-testing/defect-reporting-best-practices.md)), but overusing "auto" fixtures makes test setup less visible/traceable at the point of use.

## Common Pitfalls

- Using function/test scope for expensive setup (e.g., logging in via the UI) on every single test, drastically slowing down the suite — this is what storage-state reuse patterns exist to solve.
- Sharing session/worker-scoped mutable state (e.g., a shared cart) across tests that then interfere with each other — scope should match genuine safety for sharing, not just convenience.
- In pytest, forgetting `yield` and just `return`ing a value — this skips teardown entirely, since `yield`-based fixtures are what allow cleanup code to run after the test.
- In TypeScript, forgetting to call `await use(value)` inside a fixture — this is what actually hands control to the test; omitting it breaks the fixture chain.

## Interview Notes

- Be ready to explain fixture scope options in both ecosystems and give an example of when each scope is appropriate.
- Understand the setup → yield/use → teardown pattern in both languages — a very common question is "how do you handle cleanup after a test in your framework?"
- Be able to describe composing fixtures (fixtures depending on other fixtures) as a way to build up complex setup (e.g., authenticated + seeded-data state) from small, independent, reusable pieces.

## References

- [Pytest — Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [Playwright — pytest-playwright Fixtures](https://playwright.dev/python/docs/test-runners)
- [Playwright — Test Fixtures (Node.js)](https://playwright.dev/docs/test-fixtures)