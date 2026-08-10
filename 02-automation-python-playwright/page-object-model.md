# Page Object Model (Python & TypeScript)

## What It Is

The Page Object Model (POM) is a design pattern that encapsulates a page's (or component's) locators and interactions inside a dedicated class, separating "how to interact with the UI" from "what the test verifies." Tests call methods on page objects (`login_page.login(email, password)`) instead of interacting with locators directly, keeping test code readable and locators maintainable in one place.

## Why It Matters

- Without POM, the same locator gets duplicated across dozens of tests — a single UI change (a button's label, a field's structure) then requires updating every test file that touches it, instead of one page object.
- POM is the single most common framework architecture question in SDET interviews, because it directly demonstrates whether a candidate thinks about test *maintainability* at scale, not just writing individual passing tests.
- It's foundational to everything that follows in framework design — fixtures often *provide* page objects to tests, and larger frameworks build component objects and base page classes on top of this same principle.

## How It Works

**Core structure:**
- Each page (or reusable component) gets its own class.
- The class holds locators as attributes/properties.
- The class exposes methods representing user actions (`login()`, `add_item_to_cart()`) — not raw `.click()`/`.fill()` calls in the test itself.
- Tests read like a sequence of business actions and assertions, not low-level UI steps.

**What belongs in a page object vs. the test:**
- Page object: locators, actions, and simple state queries (`is_error_visible()`).
- Test: assertions about expected outcomes, and the sequence of business steps for that specific scenario.

Assertions are debated — some teams keep all assertions in the test file (page objects only expose state, e.g., `get_error_text()`); others allow simple visibility assertions inside the page object for convenience. Either is valid as long as it's applied consistently across the framework.

## Example

**Python:**
```python
# pages/login_page.py
from playwright.sync_api import Page

class LoginPage:
    def __init__(self, page: Page):
        self.page = page
        self.email_input = page.get_by_label("Email")
        self.password_input = page.get_by_label("Password")
        self.login_button = page.get_by_role("button", name="Log in")
        self.error_banner = page.get_by_test_id("login-error")

    def goto(self):
        self.page.goto("https://example.com/login")

    def login(self, email: str, password: str):
        self.email_input.fill(email)
        self.password_input.fill(password)
        self.login_button.click()

    def get_error_text(self) -> str:
        return self.error_banner.inner_text()
```

```python
# test_login.py
from playwright.sync_api import Page, expect
from pages.login_page import LoginPage

def test_login_fails_with_wrong_password(page: Page):
    login_page = LoginPage(page)
    login_page.goto()
    login_page.login("test@example.com", "WrongPass1")

    expect(page).to_have_url("**/login")
    assert login_page.get_error_text() == "Invalid email or password"
```

**TypeScript:**
```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorBanner: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: 'Log in' });
    this.errorBanner = page.getByTestId('login-error');
  }

  async goto() {
    await this.page.goto('https://example.com/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async getErrorText(): Promise<string> {
    return this.errorBanner.innerText();
  }
}
```

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('login fails with wrong password', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('test@example.com', 'WrongPass1');

  await expect(page).toHaveURL(/.*login/);
  expect(await loginPage.getErrorText()).toBe('Invalid email or password');
});
```

## Production Considerations

- Introduce a **BasePage** class holding shared functionality (navigation helpers, common wait logic) that other page objects extend — avoids duplicating boilerplate across every page object as the framework grows.
- Break large pages into smaller **component objects** (e.g., a `NavBar` component reused across every page) rather than one giant page object per page — this matches how modern component-based UIs (React/Vue) are actually structured.
- Decide and document the assertion placement convention (in page objects vs. tests only) once, early — inconsistency here across a growing team is a common source of framework confusion.

## Common Pitfalls

- Putting test-specific assertions inside page object methods, coupling the page object to one test's expectations and making it less reusable for other scenarios.
- Creating a single massive page object per page with dozens of unrelated locators/methods instead of breaking out reusable components (nav bar, modal, cart drawer) shared across multiple pages.
- Exposing raw Playwright locators directly from the page object instead of meaningful action methods — this defeats the purpose of POM, since tests end up doing low-level `.click()` calls on exposed locators anyway.
- Not keeping page objects and the actual UI in sync — an outdated page object with stale locators causes confusing failures that look like app bugs.

## Interview Notes

- Be ready to write a simple page object from scratch live — one of the most common practical SDET interview exercises.
- Be able to explain the trade-off around assertion placement (in page object vs. test) and state which convention you'd choose and why.
- Understand how POM evolves into component objects and a BasePage as a framework scales — this shows architectural thinking beyond a single-page example.

## References

- [Playwright — Page Object Models (Python)](https://playwright.dev/python/docs/pom)
- [Playwright — Page Object Models (Node.js)](https://playwright.dev/docs/pom)