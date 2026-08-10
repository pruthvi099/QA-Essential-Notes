# Handling Authentication (Python & TypeScript)

## What It Is

Handling authentication in Playwright means avoiding a real UI login flow on every test by authenticating once and reusing that session's **storage state** (cookies + local storage) across tests. Playwright supports saving a context's storage state to a file and loading it into new contexts, skipping the login UI entirely for tests that just need to already be logged in.

## Why It Matters

- Logging in via the UI on every test is slow and repetitive — if 100 tests each spend 3 seconds logging in, that's 5 minutes of pure overhead per run, purely on setup unrelated to what's actually being tested.
- Storage state reuse is one of the highest-leverage performance optimizations in a Playwright framework, and a very common thing SDETs are expected to have implemented, not just know about theoretically.
- It builds directly on [Browser Contexts & Test Isolation](./browser-contexts-and-test-isolation.md) — storage state is captured and restored at the context level, so understanding contexts is a prerequisite to understanding this pattern.

## How It Works

**The pattern:**
1. Log in once via the UI (or via a direct API call, which is faster) in a setup step.
2. Save the resulting context's storage state (cookies, local storage) to a JSON file.
3. In subsequent tests, create a new context using that saved storage state — the session is already authenticated, no UI login needed.

**Python** — typically implemented as a session-scoped fixture that authenticates once and yields a storage state path.

**TypeScript** — Playwright Test has first-class support for this via a global setup project that runs once before all tests, saving storage state for other projects to reuse.

## Example

**Python (session-scoped fixture pattern):**
```python
# conftest.py
import pytest
import json

@pytest.fixture(scope="session")
def storage_state_path(browser):
    context = browser.new_context()
    page = context.new_page()
    page.goto("https://example.com/login")
    page.get_by_label("Email").fill("test@example.com")
    page.get_by_label("Password").fill("Pass@123")
    page.get_by_role("button", name="Log in").click()
    page.wait_for_url("**/dashboard")

    state_path = "auth_state.json"
    context.storage_state(path=state_path)
    context.close()
    return state_path

@pytest.fixture
def authenticated_page(browser, storage_state_path):
    context = browser.new_context(storage_state=storage_state_path)
    page = context.new_page()
    yield page
    context.close()
```

```python
# test_dashboard.py — no login UI interaction needed, session already restored
def test_dashboard_visible(authenticated_page):
    authenticated_page.goto("https://example.com/dashboard")
    assert authenticated_page.get_by_test_id("username").is_visible()
```

**TypeScript (global setup project pattern):**
```typescript
// global-setup.ts
import { chromium, FullConfig } from '@playwright/test';

export default async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto('https://example.com/login');
  await page.getByLabel('Email').fill('test@example.com');
  await page.getByLabel('Password').fill('Pass@123');
  await page.getByRole('button', { name: 'Log in' }).click();
  await page.waitForURL('**/dashboard');

  await page.context().storageState({ path: 'auth-state.json' });
  await browser.close();
}
```

```typescript
// playwright.config.ts
export default defineConfig({
  globalSetup: require.resolve('./global-setup'),
  use: {
    storageState: 'auth-state.json',   // every test starts already authenticated
  },
});
```

```typescript
// tests/dashboard.spec.ts — no login UI interaction needed
import { test, expect } from '@playwright/test';

test('dashboard visible', async ({ page }) => {
  await page.goto('https://example.com/dashboard');
  await expect(page.getByTestId('username')).toBeVisible();
});
```

## Production Considerations

- Prefer authenticating via a direct API call (`page.request.post()` to a login endpoint) over a UI login flow when generating storage state — it's significantly faster and avoids the UI login flow becoming a single point of failure for every test's setup.
- For multi-role testing (admin, standard user, guest), save separate storage state files per role and select the appropriate one per test/project — don't try to force one shared authenticated state to represent multiple roles.
- Storage state files contain live session tokens — treat them as secrets in CI (don't commit them to version control, regenerate them per pipeline run or on a controlled schedule) since a leaked one is effectively a leaked session.

## Common Pitfalls

- Logging in via the full UI flow on every single test instead of reusing storage state — this is the single most common Playwright framework performance mistake in real projects.
- Sharing one storage state file across parallel test runs without considering token/session expiry — a long-lived suite run might outlast a short session timeout, causing tests to fail with "unauthenticated" errors partway through.
- Committing storage state JSON files to version control — this leaks session tokens and cookies, a real security issue, not just a hygiene one.
- Not regenerating storage state when the login flow itself changes (e.g., new required field) — stale setup code silently breaks every test relying on it.

## Interview Notes

- Be ready to explain the storage state reuse pattern end-to-end: authenticate once, save state, restore into new contexts — a very commonly asked practical Playwright framework question.
- Understand why API-based login (vs. UI-based) is preferred for generating storage state, and be able to justify it in terms of speed and reliability.
- Be able to explain how you'd handle multiple user roles needing different authenticated states in the same framework.

## References

- [Playwright — Authentication (Python)](https://playwright.dev/python/docs/auth)
- [Playwright — Authentication (Node.js)](https://playwright.dev/docs/auth)