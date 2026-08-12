# TypeScript Fundamentals for Test Automation

## What It Is

TypeScript adds static typing to JavaScript — types are checked at compile time, catching a class of bugs (wrong argument types, typos in property names, mismatched return types) before a test ever runs. This note covers the core TypeScript features that matter specifically for writing test automation code, not TypeScript as a general-purpose language.

## Why It Matters

- Type errors caught at compile time (a locator function called with the wrong argument type, a fixture used incorrectly) are far cheaper to fix than the same mistake surfacing as a confusing runtime failure mid-test-run.
- Playwright Test's own APIs are fully typed — fixtures, `Page`, `Locator`, and config objects all have proper TypeScript types, meaning your editor can autocomplete and catch mistakes in test code the same way it does in application code, which meaningfully reduces trial-and-error when writing tests.
- TypeScript proficiency is often an explicit requirement in SDET job postings for TS-based frameworks (see [QA Engineer vs. SDET](../00-start-here/qa-engineer-vs-sdet.md)) — knowing the language, not just Playwright's API, is what lets you build type-safe frameworks rather than just copy-pasting examples.

## How It Works

**Types** — annotate variables, function parameters, and return values:
```typescript
let email: string = "test@example.com";
let retries: number = 3;
let isActive: boolean = true;
```

**Interfaces** — define the shape of an object (very common for typing test data and API responses):
```typescript
interface User {
  id: number;
  email: string;
  role: 'admin' | 'customer';   // a union type, restricting to specific values
}
```

**Functions** — typed parameters and return values:
```typescript
function buildTestUser(email: string, role: 'admin' | 'customer'): User {
  return { id: Math.floor(Math.random() * 10000), email, role };
}
```

**Async/await** — Playwright's TypeScript API is entirely promise-based; nearly every Playwright method returns a `Promise` and must be awaited:
```typescript
async function loginAs(page: Page, user: User): Promise<void> {
  await page.goto('/login');
  await page.getByLabel('Email').fill(user.email);
}
```

**Generics** — write reusable functions/classes that work with multiple types while preserving type safety:
```typescript
function firstOrThrow<T>(items: T[]): T {
  if (items.length === 0) throw new Error("Array is empty");
  return items[0];
}
```

## Example

Typed test data and a type-safe helper function, the kind of code that shows up constantly in a real TypeScript automation framework:

```typescript
// types.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
}

export interface Order {
  id: number;
  items: Product[];
  total: number;
  status: 'pending' | 'shipped' | 'delivered';
}
```

```typescript
// test-data/orders.ts
import { Order, Product } from '../types';

export function buildTestOrder(items: Product[]): Order {
  const total = items.reduce((sum, item) => sum + item.price, 0);
  return {
    id: Math.floor(Math.random() * 100000),
    items,
    total,
    status: 'pending',
  };
}
```

```typescript
// tests/order.spec.ts
import { test, expect } from '@playwright/test';
import { buildTestOrder } from '../test-data/orders';
import { Product } from '../types';

test('order total calculates correctly', async ({ page }) => {
  const items: Product[] = [
    { id: 1, name: 'Mouse', price: 899, inStock: true },
    { id: 2, name: 'Keyboard', price: 1499, inStock: true },
  ];
  const order = buildTestOrder(items);

  // TypeScript catches at compile time if `order.total` were compared
  // to a string, or if `order.status` were assigned an invalid value
  expect(order.total).toBe(2398);
  expect(order.status).toBe('pending');
});
```

Contrast with what this looks like *without* types — a mistake that would only surface at runtime:
```typescript
// Without an interface, nothing stops this typo from compiling —
// it would fail silently or produce `undefined` at runtime instead
// of being caught immediately by the editor/compiler
function buildOrder(items) {
  return { total: items.reduce((sum, i) => sum + i.pryce, 0) };  // "pryce" typo
}
```

## Production Considerations

- Enable `strict` mode in `tsconfig.json` for test frameworks — it catches significantly more potential bugs (implicit `any`, null/undefined mismatches) than TypeScript's default, looser settings, and is worth the upfront friction on a codebase meant to last.
- Define shared types (test data shapes, API response shapes) once in a central `types.ts` and import them everywhere, rather than redefining similar interfaces per file — this keeps test data consistent across the framework and makes API contract changes easier to propagate.
- Avoid overusing `any` to silence type errors quickly — it defeats the purpose of using TypeScript at all and tends to accumulate into type-unsafe corners of an otherwise type-safe codebase.

## Common Pitfalls

- Forgetting `await` on Playwright's promise-based methods — TypeScript won't always catch a missing `await` at compile time (the code still "compiles"), but it causes real runtime bugs (actions firing before the previous one completes) that are hard to diagnose without knowing to look for it.
- Using loosely typed `any` for test data or API responses, losing the autocomplete and compile-time safety that's the entire point of using TypeScript in the first place.
- Not typing fixture return values explicitly (relying entirely on inference) in complex custom fixtures — explicit types make fixture usage errors show up immediately in the editor rather than requiring a run to discover.

## Interview Notes

- Be ready to explain what an interface is and why it's useful for typing test data/API responses specifically, with a concrete example.
- Understand why `await` is required on nearly every Playwright TS method, and what happens (in terms of real bugs) when it's missed.
- Be able to explain generics at a basic level with one practical test-automation example (e.g., a reusable "get first item or throw" helper) — full generic mastery isn't usually expected, but recognizing the pattern is.

## References

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Playwright — TypeScript (Node.js)](https://playwright.dev/docs/test-typescript)