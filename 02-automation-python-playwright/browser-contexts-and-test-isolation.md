# Browser Contexts & Test Isolation (Python & TypeScript)

## What It Is

A **browser context** is an isolated, incognito-like session within a single browser instance — its own cookies, local storage, and cache, independent of any other context. A single browser process can host many contexts simultaneously, each fully isolated from the others. This is the mechanism Playwright uses to guarantee **test isolation**: by default, both `pytest-playwright` and Playwright Test create a fresh context (and page) for every test.

## Why It Matters

- Test isolation is what prevents one test's leftover state (a logged-in session, items in a cart, modified local storage) from silently affecting another test — without it, test order matters and failures become hard to reproduce, exactly the kind of flakiness discussed in [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) but at the browser/session level instead of the database level.
- Contexts are much cheaper to create than full browser instances — Playwright's default pattern (one browser launched, many contexts created/destroyed per test) is what makes fast, isolated, parallel test execution practical at scale.
- Understanding contexts is necessary to correctly implement authentication reuse (see [Handling Authentication](./handling-authentication.md)) and multi-user testing scenarios (e.g., testing a chat feature between two simulated users) — both rely on deliberately creating and managing contexts beyond the default one-per-test.

## How It Works

**Hierarchy:** Browser → Browser Context → Page

- One **browser** process can be reused across many tests (expensive to launch, cheap to reuse).
- Each **context** is an isolated session (cheap to create/destroy, this is the isolation boundary).
- Each context can have multiple **pages** (tabs) — useful for testing flows involving multiple tabs/windows.

**Default behavior:** both `pytest-playwright` and Playwright Test automatically create a new context (and a `page` within it) for every single test function, and destroy it afterward — this is why tests don't need to manually clear cookies/storage between runs unless they explicitly share a context on purpose.

## Example

**Python — default isolation (no extra code needed):**
```python
# Each test automatically gets its own fresh context/page —
# no leftover cookies or storage from other tests
def test_user_a_session(page):
    page.goto("https://example.com/login")
    page.get_by_label("Email").fill("user_a@example.com")
    # ... this session is fully isolated from other tests

def test_user_b_session(page):
    page.goto("https://example.com")
    # No trace of user_a's login state here — fresh context
```

**Python — manually creating multiple contexts for a multi-user scenario:**
```python
from playwright.sync_api import Browser

def test_chat_between_two_users(browser: Browser):
    # Two separate contexts = two fully independent "users"
    context_a = browser.new_context()
    context_b = browser.new_context()

    page_a = context_a.new_page()
    page_b = context_b.new_page()

    page_a.goto("https://chat.example.com")
    page_b.goto("https://chat.example.com")

    page_a.get_by_label("Message").fill("Hello from A")
    page_a.get_by_role("button", name="Send").click()

    # Assert user B's page receives the message — proves the two
    # contexts behave as fully independent browser sessions
    assert page_b.get_by_text("Hello from A").is_visible()

    context_a.close()
    context_b.close()
```

**TypeScript — equivalent multi-user scenario:**
```typescript
import { test, expect, chromium } from '@playwright/test';

test('chat between two users', async () => {
  const browser = await chromium.launch();
  const contextA = await browser.newContext();
  const contextB = await browser.newContext();

  const pageA = await contextA.newPage();
  const pageB = await contextB.newPage();

  await pageA.goto('https://chat.example.com');
  await pageB.goto('https://chat.example.com');

  await pageA.getByLabel('Message').fill('Hello from A');
  await pageA.getByRole('button', { name: 'Send' }).click();

  await expect(pageB.getByText('Hello from A')).toBeVisible();

  await contextA.close();
  await contextB.close();
  await browser.close();
});
```

## Production Considerations

- Rely on the default per-test context isolation for the vast majority of tests — only create contexts manually when a test genuinely needs multiple simultaneous sessions (multi-user flows) or needs to reuse a saved authenticated state (see [Handling Authentication](./handling-authentication.md)).
- Reusing a single browser instance across many tests (rather than launching a new browser per test) while still getting fresh contexts per test is the standard performance pattern — browser launch is the expensive part, not context creation.
- When manually managing contexts, always close them explicitly (`context.close()`) — leaked contexts accumulate memory across a long-running test suite, especially in CI where many tests run sequentially in one worker process.

## Common Pitfalls

- Manually reusing a single context across multiple tests "to save time," reintroducing the exact state-leakage problem the default per-test isolation is designed to prevent.
- Forgetting to close manually created contexts/pages, causing memory growth over a long test run — the default fixture-managed context handles this automatically, but manual ones don't.
- Assuming a new browser is launched per test by default — it's actually the context that's fresh per test, not necessarily the browser process itself, which is often reused for performance.
- Not realizing that cookies/local storage are scoped to the context, not the browser — a common source of confusion when a manually created second context doesn't share login state with the first, even within the same test.

## Interview Notes

- Be ready to explain the Browser → Context → Page hierarchy and which layer provides test isolation — a fundamental, frequently asked Playwright architecture question.
- Understand why contexts (not full browser relaunches) are the mechanism for isolation, and why that's a deliberate performance choice.
- Be able to describe a scenario requiring multiple contexts within a single test (multi-user simulation) and explain why a single context/page couldn't achieve the same thing.

## References

- [Playwright — Browser Contexts (Python)](https://playwright.dev/python/docs/browser-contexts)
- [Playwright — Browser Contexts (Node.js)](https://playwright.dev/docs/browser-contexts)