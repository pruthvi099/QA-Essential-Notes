# Modules & Project Structure

## What It Is

TypeScript uses ES modules (`import`/`export`) to organize code across files. In a test automation framework, module structure determines how page objects, fixtures, test data factories, and utilities are organized and reused across the codebase — a well-structured project makes it obvious where new code belongs; a poorly structured one accumulates duplication and circular dependencies as it grows.

## Why It Matters

- As a framework grows past a handful of files, project structure stops being a stylistic preference and starts directly affecting maintainability — where you put a new locator, fixture, or utility should be obvious, not a judgment call made differently by every contributor.
- Barrel files (`index.ts` re-exports) and clear module boundaries reduce import clutter significantly in larger frameworks, but can also introduce circular dependency issues if used carelessly — knowing both the benefit and the risk matters.
- Interviewers assessing SDET/framework-architecture skill often ask candidates to describe how they'd structure a new TypeScript automation project from scratch — this is a direct, practical extension of the [Page Object Model](../02-automation-python-playwright/page-object-model.md) and [Fixtures](../02-automation-python-playwright/fixtures.md) concepts into full project layout.

## How It Works

**Standard `import`/`export` syntax:**
```typescript
// named export
export function buildUser() { /* ... */ }

// named import
import { buildUser } from './factories';

// default export (one per file, use sparingly in frameworks —
// named exports are generally preferred for discoverability)
export default class LoginPage { /* ... */ }
```

**A common, scalable project structure for a Playwright TypeScript framework:**
```text
project-root/
├── tests/                  # test files (*.spec.ts)
│   ├── checkout.spec.ts
│   └── login.spec.ts
├── pages/                  # page objects
│   ├── LoginPage.ts
│   └── CheckoutPage.ts
├── fixtures/                # custom fixtures
│   └── auth.fixture.ts
├── factories/                # test data factories
│   └── orderFactory.ts
├── types/                  # shared interfaces/types
│   └── index.ts
├── utils/                  # generic helpers (not page-specific)
│   └── apiClient.ts
├── playwright.config.ts
└── tsconfig.json
```

**Barrel files** — an `index.ts` that re-exports everything in a folder, simplifying imports elsewhere:
```typescript
// pages/index.ts
export * from './LoginPage';
export * from './CheckoutPage';
```
```typescript
// Instead of two separate import lines, one barrel import:
import { LoginPage, CheckoutPage } from '../pages';
```

## Example

A small but realistic slice of a structured framework, showing how the pieces from earlier notes fit into actual files:

```typescript
// types/index.ts
export interface Product {
  id: number;
  name: string;
  price: number;
}
```

```typescript
// factories/productFactory.ts
import { Product } from '../types';

export function buildProduct(overrides: Partial<Product> = {}): Product {
  return { id: Date.now(), name: 'Test Product', price: 999, ...overrides };
}
```

```typescript
// pages/CheckoutPage.ts
import { Page, Locator } from '@playwright/test';

export class CheckoutPage {
  readonly page: Page;
  readonly payButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.payButton = page.getByRole('button', { name: 'Pay' });
  }

  async goto() {
    await this.page.goto('/checkout');
  }

  async pay() {
    await this.payButton.click();
  }
}
```

```typescript
// tests/checkout.spec.ts
import { test, expect } from '@playwright/test';
import { CheckoutPage } from '../pages/CheckoutPage';
import { buildProduct } from '../factories/productFactory';

test('checkout completes for a test product', async ({ page }) => {
  const product = buildProduct({ price: 1500 });
  const checkoutPage = new CheckoutPage(page);

  await checkoutPage.goto();
  await checkoutPage.pay();
  // ... assertions using `product` data set up above
});
```

Each file has one clear responsibility, and the import paths make the dependency flow (types → factories → pages → tests) easy to follow.

## Production Considerations

- Keep `types/` free of dependencies on other framework code (pages, fixtures) — types should be the foundation everything else imports from, not the reverse; violating this direction is the most common cause of circular dependency errors.
- Use barrel files (`index.ts`) selectively for genuinely cohesive folders (all page objects, all types) — overusing them across the whole project can obscure where a given export actually lives, making "go to definition" less useful in large codebases.
- Establish the project structure early, even before many files exist — retrofitting structure onto a flat pile of test files later is far more disruptive than starting with clear folders from day one.

## Common Pitfalls

- Mixing test logic, page objects, and utilities in the same file/folder without clear separation, making the codebase harder to navigate as it grows past a handful of files.
- Creating circular dependencies (a page object importing from a fixture that itself imports that page object) — TypeScript will sometimes catch this as a compile error, but it can also cause confusing `undefined` values at runtime depending on the specific cycle.
- Overusing default exports across a large framework — named exports are generally easier to search for, refactor, and auto-import correctly across a big codebase.
- Not having a `utils/` (or similarly named) location for genuinely generic helpers, causing them to get awkwardly bolted onto unrelated page objects or duplicated across files instead.

## Interview Notes

- Be ready to sketch a scalable project structure for a new Playwright TypeScript framework from scratch — a very common framework-architecture interview question.
- Understand what a circular dependency is, how it can arise in a test framework specifically, and how clear module boundaries (types → factories → pages → tests) prevent it.
- Be able to explain the trade-off of barrel files: import convenience vs. potential structural obscurity in large codebases.

## References

- [TypeScript Handbook — Modules](https://www.typescriptlang.org/docs/handbook/2/modules.html)
- [Playwright — Project Structure best practices (Node.js)](https://playwright.dev/docs/best-practices)