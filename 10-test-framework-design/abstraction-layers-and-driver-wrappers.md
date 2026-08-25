# Abstraction Layers & Driver Wrappers

## What It Is

A driver wrapper is a thin abstraction layer sitting between test/page-object code and the underlying automation tool (Playwright, Appium, Selenium) — instead of calling `page.click()` directly everywhere, code calls a wrapper method (`driver.click()`) that internally delegates to the tool's actual API. This is the "driver/tool abstraction layer" introduced in [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md), covered here in depth.

## Why It Matters

- Without an abstraction layer, a framework is tightly coupled to one specific tool's API — if the team ever needs to switch tools (Selenium to Playwright, or add Appium alongside an existing Playwright suite), every single test and page object needs rewriting, since they all call the tool's API directly.
- A wrapper layer is also the natural place to add cross-cutting behavior consistently — custom logging on every action, automatic retry logic beyond what the tool provides natively, or standardized error handling — without scattering that logic across every individual test.
- This is a genuinely advanced framework design decision with real trade-offs (not a default "always do this") — interviewers use it to assess whether a candidate can reason about when added abstraction is worth its complexity cost, not just whether they've heard of the pattern.

## How It Works

**The core pattern:** define an interface/wrapper class with the operations your framework actually needs (click, fill, get text, wait for visible), implement it once against the underlying tool's real API, and have all higher-layer code (page objects, tests) depend on the wrapper — never the underlying tool directly.

**What typically goes in a wrapper:**
- Core actions (click, fill, navigate) delegating to the underlying tool.
- Cross-cutting concerns applied consistently: logging every action, custom retry/error-handling logic beyond the tool's defaults.
- A translation layer if truly swapping tools (e.g., supporting both Playwright and Appium behind one interface) — though this is a significant undertaking, appropriate only when genuinely justified (see Production Considerations).

## Example

**A minimal driver wrapper around Playwright, adding consistent logging as a cross-cutting concern:**
```typescript
// core/TestDriver.ts
import { Page, Locator } from '@playwright/test';

export class TestDriver {
  constructor(private page: Page, private logger: Logger) {}

  async click(locator: Locator, description: string) {
    this.logger.info(`Clicking: ${description}`);
    try {
      await locator.click();
    } catch (error) {
      this.logger.error(`Failed to click: ${description}`, error);
      throw error;
    }
  }

  async fill(locator: Locator, value: string, description: string) {
    this.logger.info(`Filling "${description}" with: ${value}`);
    await locator.fill(value);
  }

  async goto(url: string) {
    this.logger.info(`Navigating to: ${url}`);
    await this.page.goto(url);
  }
}
```

**A page object using the wrapper instead of calling Playwright's `page`/`locator` API directly:**
```typescript
// pages/LoginPage.ts
import { Page } from '@playwright/test';
import { TestDriver } from '../core/TestDriver';

export class LoginPage {
  private driver: TestDriver;

  constructor(page: Page, logger: Logger) {
    this.driver = new TestDriver(page, logger);
  }

  async login(email: string, password: string) {
    // Uses the WRAPPER, not page.getByLabel(...).fill() directly —
    // every action here is automatically logged and gets consistent
    // error handling, without the page object needing to implement it
    await this.driver.fill(this.emailField, email, 'Email field');
    await this.driver.fill(this.passwordField, password, 'Password field');
    await this.driver.click(this.loginButton, 'Login button');
  }
}
```

**What genuinely swapping the underlying tool would require** — illustrating the real cost/benefit, not just the benefit:
```typescript
// core/TestDriver.ts — if migrating from Playwright to a different tool,
// ONLY this file needs to change; every page object and test using
// TestDriver's interface remains untouched
export class TestDriver {
  // Internal implementation changes here (e.g., to a different
  // underlying library's API), but the PUBLIC interface (click, fill,
  // goto) stays the same, so nothing above this layer needs to change
}
```

## Production Considerations

- Introducing a driver wrapper is a genuine complexity trade-off — it adds an indirection layer that must be learned and maintained, and is most justified when a team has real reason to expect tool changes, wants centralized cross-cutting behavior (logging, custom retries), or is building a framework spanning multiple tools (web + mobile) that benefits from a unified interface.
- For a smaller framework using a single, stable tool with no expected changes, calling Playwright's API directly in page objects (without a wrapper) is often the simpler, equally valid choice — abstraction for its own sake, without a concrete driving need, adds cost without proportional benefit.
- If building a wrapper, keep its interface focused on what the framework actually needs — mirroring the underlying tool's entire API surface defeats the purpose (that's not abstraction, just renaming), while an interface too narrow forces frequent workarounds.

## Common Pitfalls

- Building an elaborate driver abstraction layer for a small, single-tool framework with no realistic need to swap tools — this is speculative complexity that slows development without delivering proportional value.
- Building a wrapper interface that just mirrors the underlying tool's full API one-to-one — this adds an extra layer of indirection without actually decoupling anything meaningfully.
- Not applying cross-cutting concerns (logging, error handling) consistently through the wrapper, instead scattering ad-hoc logging calls throughout individual tests — this is exactly the duplication a wrapper is meant to centralize and eliminate.
- Assuming abstraction is always a best practice regardless of context — the right answer depends on the framework's actual, concrete needs, not a blanket architectural preference.

## Interview Notes

- Be ready to explain the trade-off of introducing a driver abstraction layer — genuine benefits (tool-swap flexibility, centralized cross-cutting concerns) weighed against real added complexity — rather than presenting it as an unconditional best practice.
- Understand what specifically should live in a wrapper (cross-cutting concerns, core actions) versus what shouldn't (mirroring the entire underlying API surface).
- Be able to describe a concrete scenario where a driver wrapper's value is clear (e.g., a framework needing consistent logging across every UI action, or one anticipating a tool migration) versus one where it's unnecessary overhead.

## References

- [Martin Fowler — GatewayPattern](https://martinfowler.com/eaaCatalog/gateway.html)
- [Playwright — API Reference (Node.js)](https://playwright.dev/docs/api/class-playwright)