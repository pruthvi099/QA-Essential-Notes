# Locators (Python & TypeScript)

## What It Is

Locators are Playwright's mechanism for finding elements on a page. They are lazy and retrying by design — a locator doesn't find an element the moment it's created; it re-queries the DOM every time an action or assertion is performed on it. This is what powers Playwright's auto-waiting behavior (see [Auto-Waiting](./auto-waiting.md)).

Playwright strongly recommends **user-facing locators** (role, text, label, test-id) over CSS/XPath selectors, because they reflect how a real user or assistive technology identifies elements, and are far less likely to break when unrelated styling/markup changes.

## Why It Matters

- Locator strategy is the single biggest factor in test stability. Brittle CSS/XPath selectors tied to implementation details (`div > div:nth-child(3) > span`) break on unrelated refactors; role/test-id based locators don't.
- Both Python and TypeScript expose the identical locator API conceptually — learning the strategy once transfers directly between languages, only syntax differs.
- Interviewers frequently ask "how do you locate elements reliably" specifically to check whether a candidate defaults to brittle CSS/XPath habits carried over from Selenium, or understands Playwright's user-facing locator philosophy.

## How It Works

**Locator priority (most to least recommended):**

1. `getByRole()` — by ARIA role and accessible name (button, link, heading, etc.) — closest to how users/assistive tech perceive the page
2. `getByLabel()` — form fields by their associated `<label>`
3. `getByPlaceholder()` — input fields by placeholder text
4. `getByText()` — elements by visible text content
5. `getByTestId()` — by a dedicated `data-testid` attribute — stable but requires the app to add these attributes
6. CSS / XPath — last resort, only when none of the above apply (e.g., targeting a specific SVG path)

Locators are chainable and filterable (`.filter()`, `.first`, `.nth()`, `.last`) to narrow down matches without resorting to complex CSS.

## Example

**Python:**
```python
from playwright.sync_api import Page

def test_locator_strategies(page: Page):
    page.goto("https://example.com/login")

    # Role-based (preferred)
    page.get_by_role("button", name="Log in").click()

    # Label-based
    page.get_by_label("Email").fill("test@example.com")

    # Placeholder-based
    page.get_by_placeholder("Enter password").fill("Pass@123")

    # Text-based
    page.get_by_text("Forgot password?").click()

    # Test-id based
    page.get_by_test_id("login-error-banner").is_visible()

    # Filtering a list down to one match
    page.get_by_role("listitem").filter(has_text="Wireless Mouse").get_by_role("button", name="Remove").click()
```

**TypeScript:**
```typescript
import { test, expect } from '@playwright/test';

test('locator strategies', async ({ page }) => {
  await page.goto('https://example.com/login');

  // Role-based (preferred)
  await page.getByRole('button', { name: 'Log in' }).click();

  // Label-based
  await page.getByLabel('Email').fill('test@example.com');

  // Placeholder-based
  await page.getByPlaceholder('Enter password').fill('Pass@123');

  // Text-based
  await page.getByText('Forgot password?').click();

  // Test-id based
  await expect(page.getByTestId('login-error-banner')).toBeVisible();

  // Filtering a list down to one match
  await page.getByRole('listitem').filter({ hasText: 'Wireless Mouse' }).getByRole('button', { name: 'Remove' }).click();
});
```

The API is nearly 1:1 between languages (`get_by_role` ↔ `getByRole`, `get_by_test_id` ↔ `getByTestId`) — only casing conventions differ (snake_case vs. camelCase), which makes moving between the two ecosystems straightforward once the strategy is understood.

## Production Considerations

- Coordinate with developers to add `data-testid` attributes to elements that don't have a stable role/label/text (icon-only buttons, dynamic widgets) — this is a cross-team dependency worth raising early in a framework's life, not retrofitting later.
- Avoid locators tied to CSS classes used for styling (`.btn-primary`) — these change with design updates unrelated to functionality, causing unnecessary test breakage.
- Prefer scoping locators within a container (`page.get_by_role("region", name="Cart").get_by_role("button", name="Remove")`) over global page-wide locators when a page has repeated similar elements — this avoids ambiguous "strict mode violation" errors when multiple matches exist.

## Common Pitfalls

- Defaulting to CSS/XPath selectors out of Selenium habit, missing Playwright's more stable, purpose-built locator API.
- Using `getByText()` for interactive elements (buttons, links) instead of `getByRole()` — text-based locators break when copy changes for localization or wording tweaks, while role + accessible name is more stable.
- Not handling Playwright's "strict mode" — locators that match more than one element throw an error by default; use `.first`, `.nth()`, or better filtering instead of assuming a single match.
- Overusing `getByTestId()` everywhere by default — it's reliable, but skips validating that the app is actually accessible (role/label-based locators double as a lightweight accessibility check, since if a locator can't find an element by role, a screen reader user likely can't either).

## Interview Notes

- Be ready to explain Playwright's locator priority list and *why* role-based locators are preferred over CSS/XPath — this is one of the most commonly asked Playwright-specific questions.
- Understand "strict mode" and how to resolve a multiple-match error without resorting to a brittle CSS selector.
- Be able to connect locator strategy to accessibility — using role/label-based locators as an implicit accessibility signal is a strong, less commonly known talking point.

## References

- [Playwright — Locators (Python)](https://playwright.dev/python/docs/locators)
- [Playwright — Locators (Node.js)](https://playwright.dev/docs/locators)
- [Playwright — Best Practices](https://playwright.dev/docs/best-practices)