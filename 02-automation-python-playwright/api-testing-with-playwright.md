# API Testing with Playwright (Python & TypeScript)

## What It Is

Playwright includes a built-in HTTP client — `APIRequestContext` — for making direct API calls without a browser. This lets you write pure API tests inside the same framework, and combine API calls with UI tests (e.g., seeding data via API before testing a UI flow, or verifying a UI action's effect via a direct API check), without needing a separate HTTP library.

## Why It Matters

- API tests are faster and more stable than UI tests for verifying backend logic — using Playwright's built-in client means one framework/toolchain covers both UI and API testing, instead of maintaining a separate library (`requests` in Python, `axios` in TS) with its own auth/config setup.
- Combining API and UI in the same test is a common, high-value pattern: seed test data via a fast API call instead of clicking through the UI to create it, then test the actual UI flow you care about — this directly reduces test setup time and flakiness.
- Interviewers often ask how you'd verify a UI action actually persisted correctly on the backend — using `APIRequestContext` to check the database/API state after a UI action is the expected, practical answer.

## How It Works

`APIRequestContext` supports all standard HTTP methods (`get`, `post`, `put`, `patch`, `delete`), handles JSON serialization automatically, and — critically — **shares cookies/auth state with a browser context** when created from one, meaning an API call made after a UI login is already authenticated.

**Two ways to use it:**
1. **`page.request`** — makes API calls using the same context as the current page, sharing cookies/session automatically.
2. **A standalone `APIRequestContext`** (via `request.new_context()` / `playwright.request.newContext()`) — for pure API tests with no browser/page involved at all, faster since no browser is launched.

## Example

**Python — seeding data via API, then testing the UI:**
```python
from playwright.sync_api import Page

def test_order_appears_in_order_history(page: Page):
    # Fast setup: create an order via API instead of clicking through checkout
    response = page.request.post("/api/orders", data={
        "items": [{"product_id": 101, "qty": 1}],
    })
    assert response.status == 201
    order_id = response.json()["id"]

    # Now test the actual UI behavior we care about
    page.goto("/account/orders")
    assert page.get_by_test_id(f"order-{order_id}").is_visible()
```

**Python — pure API test with a standalone context:**
```python
import pytest
from playwright.sync_api import APIRequestContext, playwright as pw

@pytest.fixture(scope="session")
def api_context(playwright):
    context = playwright.request.new_context(
        base_url="https://api.example.com",
        extra_http_headers={"Authorization": "Bearer test-token"},
    )
    yield context
    context.dispose()

def test_get_orders_returns_200(api_context: APIRequestContext):
    response = api_context.get("/orders")
    assert response.status == 200
    assert isinstance(response.json(), list)
```

**TypeScript — equivalent patterns:**
```typescript
import { test, expect } from '@playwright/test';

test('order appears in order history', async ({ page }) => {
  const response = await page.request.post('/api/orders', {
    data: { items: [{ productId: 101, qty: 1 }] },
  });
  expect(response.status()).toBe(201);
  const { id: orderId } = await response.json();

  await page.goto('/account/orders');
  await expect(page.getByTestId(`order-${orderId}`)).toBeVisible();
});
```

```typescript
import { test, expect, request, APIRequestContext } from '@playwright/test';

let apiContext: APIRequestContext;

test.beforeAll(async () => {
  apiContext = await request.newContext({
    baseURL: 'https://api.example.com',
    extraHTTPHeaders: { Authorization: 'Bearer test-token' },
  });
});

test.afterAll(async () => {
  await apiContext.dispose();
});

test('get orders returns 200', async () => {
  const response = await apiContext.get('/orders');
  expect(response.status()).toBe(200);
  expect(Array.isArray(await response.json())).toBeTruthy();
});
```

## Production Considerations

- Prefer API-based test data setup over UI-based setup wherever possible — it's dramatically faster and removes UI flakiness from the *setup* portion of a test that's actually meant to verify something else entirely.
- Use a standalone `APIRequestContext` (not `page.request`) for pure API test suites that never touch the UI — this avoids the overhead of launching a browser at all, making pure API suites significantly faster to run.
- Always dispose of manually created API contexts (`context.dispose()` / `apiContext.dispose()`) to release resources cleanly, same principle as closing manually created browser contexts (see [Browser Contexts](./browser-contexts-and-test-isolation.md)).

## Common Pitfalls

- Using a separate HTTP library (`requests`/`axios`) alongside Playwright instead of its built-in `APIRequestContext`, duplicating auth/config setup unnecessarily when one tool already covers both needs.
- Building all test data through the UI even when a fast API call would do the same setup in a fraction of the time — this is one of the most common, avoidable sources of slow test suites.
- Forgetting that `page.request` shares the page's current session — using it expecting a clean, unauthenticated request when the page is already logged in produces confusing, unintended results.
- Not asserting on response status codes explicitly before parsing the body — a failed request (4xx/5xx) with an unexpected body shape causes a confusing JSON-parsing error instead of a clear, actionable status-code assertion failure.

## Interview Notes

- Be ready to explain the API+UI combination pattern (seed via API, verify via UI, or the reverse) and why it's preferred over doing everything through the UI.
- Understand the difference between `page.request` (shares session) and a standalone `APIRequestContext` (independent, faster, no browser needed).
- Be able to describe how you'd structure a pure API test suite using Playwright's built-in client instead of introducing a separate HTTP library.

## References

- [Playwright — API Testing (Python)](https://playwright.dev/python/docs/api-testing)
- [Playwright — API Testing (Node.js)](https://playwright.dev/docs/api-testing)