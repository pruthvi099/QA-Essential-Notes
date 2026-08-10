# Auto-Waiting (Python & TypeScript)

## What It Is

Auto-waiting is Playwright's built-in behavior of automatically waiting for an element to be "actionable" before performing an action on it — no manual `sleep()` or explicit wait calls required for most cases. Before clicking, filling, or checking an element, Playwright waits for a series of actionability checks to pass: the element is attached to the DOM, visible, stable (not animating), enabled, and not obscured by another element.

## Why It Matters

- Auto-waiting is what eliminates most of the flaky, timing-based failures that plagued older tools (Selenium) requiring manual explicit waits everywhere. Understanding *what* it waits for (and what it doesn't) is essential to writing stable tests and debugging failures correctly.
- Misunderstanding auto-waiting is one of the most common sources of both flaky tests (assuming it waits for something it doesn't, like a network response) and unnecessarily slow tests (adding redundant manual waits on top of it).
- This is a frequent interview topic specifically because it differentiates Playwright from legacy tools — being able to explain the actionability checks precisely shows real hands-on Playwright experience, not just surface familiarity.

## How It Works

Before most actions, Playwright automatically waits for the target element to be:

1. **Attached** — present in the DOM
2. **Visible** — has non-zero size and no `visibility: hidden` / `display: none`
3. **Stable** — not still animating (e.g., mid-transition)
4. **Enabled** — not disabled (relevant for form elements)
5. **Receives events** — not obscured by another element on top of it (e.g., a loading overlay)
6. **Editable** (for fill/type actions only) — not readonly

If these conditions aren't met within the timeout, the action fails with a clear, specific error stating which check failed — a major debugging advantage over generic "element not found" errors.

**What auto-waiting does NOT cover:**
- Waiting for a network request/response to complete (use `page.wait_for_response()` / `page.waitForResponse()` explicitly)
- Waiting for arbitrary custom conditions unrelated to element state (use `page.wait_for_function()` / `page.waitForFunction()`)
- Assertions (`expect()`) have their own separate retrying mechanism, covered in [Assertions](./assertions.md)

## Example

**Python:**
```python
from playwright.sync_api import Page

def test_add_to_cart_with_auto_waiting(page: Page):
    page.goto("https://shop.example.com/products/101")

    # Playwright automatically waits for this button to be visible,
    # stable, and enabled before clicking — no manual wait needed
    page.get_by_role("button", name="Add to Cart").click()

    # This waits for the cart badge to actually update, since
    # get_by_text re-queries until it finds matching visible text
    page.get_by_test_id("cart-badge").click()

    # Explicit wait needed here: waiting for a NETWORK response,
    # which auto-waiting does not cover
    with page.expect_response("**/api/cart/items") as response_info:
        page.get_by_role("button", name="Refresh Cart").click()
    assert response_info.value.status == 200
```

**TypeScript:**
```typescript
import { test, expect } from '@playwright/test';

test('add to cart with auto-waiting', async ({ page }) => {
  await page.goto('https://shop.example.com/products/101');

  // Auto-waits for visible, stable, enabled before clicking
  await page.getByRole('button', { name: 'Add to Cart' }).click();

  await page.getByTestId('cart-badge').click();

  // Explicit wait for a network response — not covered by auto-waiting
  const responsePromise = page.waitForResponse('**/api/cart/items');
  await page.getByRole('button', { name: 'Refresh Cart' }).click();
  const response = await responsePromise;
  expect(response.status()).toBe(200);
});
```

## Production Considerations

- Avoid `page.wait_for_timeout()` / `page.waitForTimeout()` (a hardcoded sleep) as a fix for flaky tests — it's almost always masking a real actionability or timing issue that a proper wait (`wait_for_response`, `wait_for_selector` with a specific state, or a retrying assertion) should handle instead.
- When auto-waiting fails, Playwright's error message states exactly which actionability check failed (e.g., "element is not visible") — read this message before adding manual waits; it usually points directly at the real underlying issue (an overlay blocking the element, an animation still running).
- For custom loading indicators or app-specific "ready" states, `wait_for_function()` / `waitForFunction()` lets you wait on arbitrary JS conditions — useful for apps with non-standard loading patterns auto-waiting can't infer.

## Common Pitfalls

- Adding `sleep()` / hardcoded timeouts "just in case" alongside auto-waiting — this makes tests slower without actually fixing the underlying flakiness, and often just narrowly hides a real race condition instead of resolving it.
- Assuming auto-waiting covers network requests completing — a common source of flaky tests where an action succeeds (element was clickable) but the resulting data hasn't loaded yet, since that requires an explicit wait.
- Misreading actionability failures as "element not found" bugs in the app, when the real issue is something like a modal overlay intercepting clicks — the specific Playwright error message usually clarifies this if read carefully.

## Interview Notes

- Be ready to list the actionability checks Playwright performs before an action (attached, visible, stable, enabled, receives events) — a very common specific interview question.
- Be able to explain clearly what auto-waiting does NOT cover (network responses, custom async conditions) and how to handle those cases explicitly.
- Understand why hardcoded sleeps are considered an anti-pattern in Playwright specifically, given auto-waiting already handles most timing issues — this shows you understand the tool's design philosophy, not just its API surface.

## References

- [Playwright — Auto-waiting (Python)](https://playwright.dev/python/docs/actionability)
- [Playwright — Auto-waiting (Node.js)](https://playwright.dev/docs/actionability)