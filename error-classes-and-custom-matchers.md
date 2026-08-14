# Custom Matchers & Test Reporting Types

## What It Is

Playwright Test's `expect` can be extended with **custom matchers** — assertions that encapsulate a domain-specific check (e.g., `toBeValidOrder()`, `toHaveDiscountApplied()`) behind a readable, reusable name. This note covers writing custom matchers, and typing custom test annotations/metadata that feed into Playwright's HTML reporter — both TypeScript-specific extension points that don't have a direct Python-Playwright parallel in the same form.

## Why It Matters

- Repeating the same multi-line assertion logic across many tests (e.g., checking several properties of an "order" object) is harder to read and maintain than a single, well-named custom matcher — `expect(order).toBeValidOrder()` communicates intent far more clearly than five separate assertions inline.
- Custom matchers centralize assertion logic in one place — when the definition of "valid" changes, one matcher gets updated instead of hunting down every test that duplicated the check inline.
- This is a more advanced framework-extension topic that shows up in senior/SDET-level interviews specifically — it demonstrates comfort extending the testing tool itself, not just using its built-in API.

## How It Works

Custom matchers are added via `expect.extend()`, and must be typed correctly so TypeScript recognizes the new matcher on `expect()` calls throughout the codebase — this requires augmenting Playwright's `Matchers` interface via TypeScript's declaration merging.

**Steps:**
1. Write the matcher function (returns `{ pass: boolean, message: () => string }`).
2. Register it with `expect.extend()`.
3. Declare its TypeScript type via an ambient `.d.ts` file so `expect(x).yourMatcher()` type-checks correctly everywhere.

## Example

A custom matcher for validating an order object's shape and values, plus its type declaration:

```typescript
// matchers/toBeValidOrder.ts
import { expect as baseExpect } from '@playwright/test';
import { Order } from '../types';

export const expect = baseExpect.extend({
  toBeValidOrder(received: Order) {
    const errors: string[] = [];

    if (received.id <= 0) errors.push('id must be positive');
    if (received.items.length === 0) errors.push('items must not be empty');
    if (received.total !== received.items.reduce((s, i) => s + i.price, 0)) {
      errors.push('total does not match sum of item prices');
    }

    const pass = errors.length === 0;
    return {
      pass,
      message: () =>
        pass
          ? 'expected order NOT to be valid'
          : `expected order to be valid, but found issues: ${errors.join(', ')}`,
    };
  },
});
```

```typescript
// matchers/types.d.ts — makes TypeScript recognize the new matcher
import { Order } from '../types';

declare module '@playwright/test' {
  interface Matchers<R, T> {
    toBeValidOrder(this: { isNot: boolean }, ...args: T extends Order ? [] : never): R;
  }
}
```

```typescript
// tests/order.spec.ts
import { test } from '@playwright/test';
import { expect } from '../matchers/toBeValidOrder';
import { buildOrder, buildProduct } from '../factories';

test('order is valid after checkout', async () => {
  const order = buildOrder({ items: [buildProduct({ price: 500 })], total: 500 });

  // Reads as a single, clear intent instead of three separate
  // inline assertions repeated across every order-related test
  expect(order).toBeValidOrder();
});
```

## Production Considerations

- Reserve custom matchers for checks that are genuinely repeated across many tests with meaningful domain logic (like "is this order valid") — for a one-off check used in a single test, a plain `expect()` call is simpler and doesn't need the maintenance overhead of a matcher definition.
- Keep the `.d.ts` type declaration in sync with the matcher implementation — a mismatch (matcher accepts different arguments than the type declares) causes confusing compile errors or, worse, `any`-typed matcher calls that silently lose type safety.
- Custom matchers should produce clear, specific failure messages (as shown above, listing exactly which checks failed) — a vague "expected order to be valid" without detail defeats the debugging value the matcher was meant to add.

## Common Pitfalls

- Writing a custom matcher without also writing its TypeScript type declaration — the matcher works at runtime but `expect(x).toBeValidOrder()` shows a compile error elsewhere in the codebase, or worse, silently type-checks as `any`.
- Overusing custom matchers for simple, single-use checks — this adds indirection and a file to maintain for something a plain inline assertion would express just as clearly.
- Writing a matcher's failure message vaguely instead of including the specific reason(s) it failed — this reintroduces the same debugging friction custom matchers are meant to eliminate.
- Forgetting that the custom `expect` (from the extended module) must be imported instead of the default `@playwright/test` `expect` in any file that uses the new matcher — using the wrong import causes a "matcher not found" error.

## Interview Notes

- Be ready to explain when a custom matcher is worth writing versus when a plain assertion is simpler and sufficient — restraint here matters as much as knowing the syntax.
- Understand the two-part nature of adding a custom matcher in TypeScript: the runtime implementation (`expect.extend()`) and the type declaration (`declare module`) — many candidates know one half but not the other.
- Be able to describe a real domain-specific check from a project you've worked on (or a plausible one) that would benefit from being expressed as a custom matcher.

## References

- [Playwright — Custom Expect Matchers (Node.js)](https://playwright.dev/docs/test-assertions#custom-expect-message)
- [TypeScript Handbook — Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)