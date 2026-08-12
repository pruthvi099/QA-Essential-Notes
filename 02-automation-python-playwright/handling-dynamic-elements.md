# Handling Dynamic Elements (Python & TypeScript)

## What It Is

Dynamic elements are UI components that don't behave like a plain, static DOM element — iframes (embedded documents with their own DOM), shadow DOM (encapsulated component internals, common in web components/design systems), and content that loads asynchronously after the initial page render. Playwright provides dedicated APIs for each, since standard locators can't reach into an iframe or a closed shadow root without them.

## Why It Matters

- Real-world apps frequently embed third-party content (payment widgets, chat widgets, ads) via iframes, and increasingly use shadow DOM through component libraries (many design systems compile to web components) — not knowing how to target these correctly is a common blocker in real automation work, not just an edge case.
- A locator that silently fails to find an element inside an iframe (because it was never told to look inside the iframe) produces a confusing "element not found" error that looks like a flaky test but is actually a fundamental targeting mistake.
- This is one of the more advanced, practically differentiating topics in interviews — many candidates can write a basic locator but haven't dealt with iframes/shadow DOM, so fluency here signals real hands-on experience with non-trivial applications.

## How It Works

**Iframes** — Playwright locators do NOT automatically search inside iframes. You must explicitly get a `FrameLocator` first, then locate within it.

**Shadow DOM** — Playwright's locators (`get_by_role`, `get_by_text`, etc.) **do** pierce open shadow roots automatically — no special API needed for open shadow DOM, unlike iframes. Closed shadow roots (rare, deliberately inaccessible by design) cannot be accessed at all, by browser design, not a Playwright limitation.

**Dynamically loaded content** — handled via the same auto-waiting/web-first assertion mechanisms already covered (see [Auto-Waiting](./auto-waiting.md), [Assertions](./assertions.md)) — Playwright waits for the element to appear, so no special API is usually needed beyond correct locators and assertions.

## Example

**Python — targeting an iframe:**
```python
from playwright.sync_api import Page

def test_payment_iframe(page: Page):
    page.goto("https://shop.example.com/checkout")

    # Must explicitly target the iframe first — a plain page.get_by_label()
    # would NOT find elements inside it
    payment_frame = page.frame_locator("iframe[name='payment-widget']")
    payment_frame.get_by_label("Card Number").fill("4242424242424242")
    payment_frame.get_by_label("Expiry").fill("12/28")
    payment_frame.get_by_role("button", name="Pay").click()
```

**Python — shadow DOM (works automatically, no special API):**
```python
def test_shadow_dom_component(page: Page):
    page.goto("https://shop.example.com/product/101")

    # If <custom-rating-widget> uses an OPEN shadow root internally,
    # this just works — Playwright pierces shadow DOM by default
    page.get_by_role("button", name="5 stars").click()
```

**Python — dynamically loaded content (just needs correct auto-waiting):**
```python
def test_lazy_loaded_reviews(page: Page):
    page.goto("https://shop.example.com/product/101")
    page.get_by_role("button", name="Load Reviews").click()

    # Auto-waiting handles the delay — no manual wait/sleep needed
    review = page.get_by_text("Great product, fast shipping!")
    review.scroll_into_view_if_needed()
    assert review.is_visible()
```

**TypeScript — equivalent patterns:**
```typescript
import { test, expect } from '@playwright/test';

test('payment iframe', async ({ page }) => {
  await page.goto('https://shop.example.com/checkout');

  const paymentFrame = page.frameLocator("iframe[name='payment-widget']");
  await paymentFrame.getByLabel('Card Number').fill('4242424242424242');
  await paymentFrame.getByLabel('Expiry').fill('12/28');
  await paymentFrame.getByRole('button', { name: 'Pay' }).click();
});

test('shadow DOM component', async ({ page }) => {
  await page.goto('https://shop.example.com/product/101');
  await page.getByRole('button', { name: '5 stars' }).click();
});

test('lazy loaded reviews', async ({ page }) => {
  await page.goto('https://shop.example.com/product/101');
  await page.getByRole('button', { name: 'Load Reviews' }).click();

  const review = page.getByText('Great product, fast shipping!');
  await review.scrollIntoViewIfNeeded();
  await expect(review).toBeVisible();
});
```

## Production Considerations

- For third-party iframes (payment providers, embedded widgets) you don't control, coordinate with the provider's testing documentation — some providers offer specific test card numbers/test modes designed for automated testing (as shown in the payment example above).
- Nested iframes (an iframe within an iframe) require chaining `frame_locator()` / `frameLocator()` calls — this is straightforward but easy to miss when first debugging a "not found" error inside deeply embedded content.
- Closed shadow roots are intentionally inaccessible by browser design (not a workaround-able Playwright limitation) — if a critical flow lives entirely inside one, that's a conversation with developers about testability, not something to solve purely from the test side.

## Common Pitfalls

- Trying to locate an element inside an iframe with a plain `page.get_by_...()` call instead of first going through `frame_locator()` / `frameLocator()` — this is the most common iframe-related mistake and produces a misleading "not found" error.
- Assuming shadow DOM needs special handling like iframes do — for open shadow roots (the common case), it doesn't; over-engineering a workaround for something Playwright already handles is wasted effort.
- Not realizing dynamically loaded content usually needs zero special handling beyond correct locators and auto-waiting — reaching for manual waits/sleeps here is unnecessary in most cases.
- Forgetting `scroll_into_view_if_needed()` / `scrollIntoViewIfNeeded()` for content below the fold that only loads/renders once scrolled into view (common with lazy-loaded images/content) — actions on off-screen elements can behave unexpectedly without it.

## Interview Notes

- Be ready to explain why iframes need `frame_locator()`/`frameLocator()` explicitly while shadow DOM (open) doesn't need special handling — this distinction is a common, specific interview question.
- Understand that closed shadow roots are inaccessible by design, not a tooling gap — knowing this distinction shows genuine hands-on debugging experience versus surface-level API familiarity.
- Be able to describe how you'd approach automating a flow embedded in a third-party iframe you don't control (e.g., a payment widget) — a common real-world scenario question.

## References

- [Playwright — Frames (Python)](https://playwright.dev/python/docs/frames)
- [Playwright — Frames (Node.js)](https://playwright.dev/docs/frames)