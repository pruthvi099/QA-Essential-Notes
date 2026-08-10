# Assertions (Python & TypeScript)

## What It Is

Playwright provides **web-first assertions** via `expect()` — assertions built specifically for the async, dynamic nature of web UIs. Unlike a plain equality check, web-first assertions automatically **retry** until the expected condition is met or a timeout is reached, matching the same philosophy as auto-waiting (see [Auto-Waiting](./auto-waiting.md)).

This is distinct from a plain assertion (`assert x == y`), which checks a value exactly once, at the moment it's evaluated, with no retrying.

## Why It Matters

- Using plain assertions against dynamic UI state is one of the most common causes of flaky tests — the value is checked before the UI has finished updating, and the assertion fails even though the app is working correctly (it just hadn't updated *yet*).
- Web-first assertions solve this at the framework level: they poll the actual condition until it's true or the timeout expires, removing the need for manual retry loops or sleeps before assertions.
- Interviewers use assertion questions to check whether a candidate understands *why* Playwright's `expect()` exists as a distinct API from a language's built-in assertion — conflating the two is a common sign of surface-level Playwright knowledge.

## How It Works

**Common web-first assertions:**

| Assertion | Checks |
|---|---|
| `to_be_visible()` / `toBeVisible()` | Element is visible |
| `to_have_text()` / `toHaveText()` | Element's text content matches |
| `to_have_value()` / `toHaveValue()` | Input's value matches |
| `to_be_enabled()` / `toBeEnabled()` | Element is enabled |
| `to_have_count()` / `toHaveCount()` | Locator matches an expected number of elements |
| `to_have_url()` / `toHaveURL()` | Page URL matches |
| `to_have_title()` / `toHaveTitle()` | Page title matches |

Each of these polls the underlying condition repeatedly (default 5s timeout, configurable) rather than checking once — this is the core behavioral difference from a plain `assert`.

**When a plain assertion is still correct to use:** for values that are already stable and don't depend on UI timing — e.g., asserting on a value returned directly from an API response object, or a computed value in a helper function. Web-first assertions exist specifically for UI state that changes asynchronously.

## Example

**Python:**
```python
from playwright.sync_api import Page, expect

def test_add_to_cart_updates_badge(page: Page):
    page.goto("https://shop.example.com/products/101")
    page.get_by_role("button", name="Add to Cart").click()

    # Web-first assertion: retries until the badge shows "1",
    # tolerating the brief delay between click and UI update
    expect(page.get_by_test_id("cart-badge")).to_have_text("1")

    # Plain assert is fine here — checking a value already
    # returned synchronously, no UI timing involved
    item_count = len(page.get_by_role("listitem").all())
    assert item_count >= 0
```

**TypeScript:**
```typescript
import { test, expect } from '@playwright/test';

test('add to cart updates badge', async ({ page }) => {
  await page.goto('https://shop.example.com/products/101');
  await page.getByRole('button', { name: 'Add to Cart' }).click();

  // Web-first assertion: retries until the badge shows "1"
  await expect(page.getByTestId('cart-badge')).toHaveText('1');

  // Plain assert (Node's built-in or a library) is fine for
  // non-UI, already-resolved values
  const itemCount = await page.getByRole('listitem').count();
  expect(itemCount).toBeGreaterThanOrEqual(0);
});
```

A common mistake, shown for contrast — using a plain assertion against live UI state instead of the web-first version:

```python
# FLAKY: reads the text ONCE immediately after the click, before
# the UI has necessarily finished updating — no retrying
badge_text = page.get_by_test_id("cart-badge").text_content()
assert badge_text == "1"   # fails intermittently under real timing
```

## Production Considerations

- Prefer web-first assertions (`expect(locator)...`) over reading a value and asserting on it separately — the retrying behavior is lost the moment you extract the value out of the locator chain.
- Configure a sensible global assertion timeout in `pytest.ini` / `playwright.config.ts` rather than scattering per-assertion timeout overrides — reserve per-assertion timeouts for genuinely slower-than-usual specific operations.
- `to_have_count()` / `toHaveCount()` is the correct way to assert on a dynamic list's length (e.g., "cart has 3 items") — it retries as items load in, unlike checking `.length` on a plain array snapshot.

## Common Pitfalls

- Extracting a value with `.text_content()` / `.textContent()` and asserting on it with a plain `assert`/`expect` from a testing library other than Playwright's — this discards the retry behavior and reintroduces flakiness.
- Assuming `expect()` in Playwright behaves like a generic assertion library's `expect()` — Playwright's is specifically retrying for locator-based assertions; non-locator assertions (numbers, strings) still work but don't retry, since there's nothing to poll.
- Setting assertion timeouts too high globally "to reduce flakiness" — this just makes genuinely broken tests take much longer to fail, slowing feedback without fixing the root cause.

## Interview Notes

- Be ready to explain the core difference between a web-first assertion and a plain language assertion — specifically, retrying behavior — this is one of the most fundamental Playwright concepts asked about.
- Be able to identify, given a code snippet, whether an assertion will be flaky (extracting a value then asserting) vs. stable (asserting directly on the locator).
- Understand that `expect()` in Playwright Test (TS) and `expect()` from `playwright.sync_api` / `pytest-playwright` (Python) are the same concept, API-aligned across languages — not to be confused with `assert` (Python) or `chai`/`jest` assertions used elsewhere.

## References

- [Playwright — Assertions (Python)](https://playwright.dev/python/docs/test-assertions)
- [Playwright — Assertions (Node.js)](https://playwright.dev/docs/test-assertions)