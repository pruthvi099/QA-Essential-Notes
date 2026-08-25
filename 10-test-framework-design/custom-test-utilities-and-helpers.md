# Custom Test Utilities & Helpers

## What It Is

This note covers the utility/helper layer of a test framework (see [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md)) — genuinely reusable functions not tied to any specific page or test (date formatting, random data generation, retry helpers, common assertions) — and the judgment involved in building a useful utility library without over-engineering it into unnecessary complexity.

## Why It Matters

- Utility functions are where duplication most commonly creeps into a growing test suite — the same "generate a random email," "wait for a condition," or "format a currency value" logic reimplemented slightly differently across many test files, each with its own subtle bugs.
- A well-curated utility library compounds in value as a framework grows — but a poorly curated one (utilities that are too specific, too generic, or duplicated with slightly different behavior) becomes its own maintenance burden, sometimes worse than the duplication it was meant to prevent.
- This is a practical, everyday framework contribution — deciding "does this deserve to be a shared utility, or does it stay local to this one test" is a judgment call SDETs make constantly, and doing it well is a real, differentiating skill.

## How It Works

**What makes a good candidate for a shared utility:**
- Genuinely reusable across multiple, unrelated tests/pages (not just theoretically reusable — actually used more than once).
- Self-contained, with no dependency on specific page objects or test scenarios (see the dependency-direction principle in [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md)).
- Solves a real, recurring need — data generation, common formatting, retry/wait logic beyond what the automation tool provides natively.

**When NOT to extract a utility:**
- A piece of logic used exactly once, with no clear signal it'll be needed again — premature extraction adds indirection without payoff (the same "don't over-generalize" principle from [Generics & Reusable Test Utilities](../03-typescript-playwright/generics-and-reusable-test-utilities.md)).
- Logic that's actually specific to one page's behavior — that belongs as a method on that page object, not a generic utility.

## Example

A focused, well-curated utility module — each function genuinely reusable, self-contained, and solving a recurring need:

```typescript
// utils/test-data-generators.ts
export function generateUniqueEmail(prefix = 'test'): string {
  return `${prefix}_${Date.now()}_${Math.random().toString(36).slice(2, 8)}@example.com`;
}

export function generateRandomOrderId(): number {
  return Math.floor(Math.random() * 1_000_000);
}
```

```typescript
// utils/formatting.ts
export function formatCurrency(amount: number, currency = 'INR'): string {
  return new Intl.NumberFormat('en-IN', { style: 'currency', currency }).format(amount);
}

export function parseCurrencyString(formatted: string): number {
  return parseFloat(formatted.replace(/[^0-9.-]+/g, ''));
}
```

```typescript
// utils/retry.ts — a generic retry helper beyond what Playwright's
// own auto-waiting covers, for non-locator-based async conditions
export async function retryUntil<T>(
  fn: () => Promise<T>,
  condition: (result: T) => boolean,
  { maxAttempts = 5, delayMs = 1000 } = {}
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const result = await fn();
    if (condition(result)) return result;
    if (attempt < maxAttempts) await new Promise(r => setTimeout(r, delayMs));
  }
  throw new Error(`Condition not met after ${maxAttempts} attempts`);
}
```

Usage across multiple, unrelated tests — demonstrating genuine reuse, the real justification for extraction:

```typescript
import { generateUniqueEmail } from '../utils/test-data-generators';
import { formatCurrency } from '../utils/formatting';
import { retryUntil } from '../utils/retry';

test('checkout with a new customer', async ({ page, request }) => {
  const email = generateUniqueEmail('checkout');   // reused across MANY tests
  // ...
});

test('order total displays correctly', async ({ page }) => {
  const expectedTotal = formatCurrency(1500);
  await expect(page.getByTestId('order-total')).toHaveText(expectedTotal);
});

test('waits for async report generation to complete', async ({ request }) => {
  const report = await retryUntil(
    () => request.get('/api/reports/latest').then(r => r.json()),
    (result) => result.status === 'complete'
  );
  expect(report.data).toBeDefined();
});
```

A counter-example — logic that was extracted prematurely and shouldn't have been:
```typescript
// BAD: this is used in exactly ONE test, is tightly coupled to one
// specific page's exact structure, and provides no real reuse benefit —
// it should just be inline in that one test, or a method on that
// page object, not a generic "utility"
export function fillOutTheSpecificCheckoutFormForTestScenarioB(page: Page) {
  // ...
}
```

## Production Considerations

- Periodically audit the utility library for functions that are only ever called from one place — these are candidates for either inlining back into their single caller, or for genuinely confirming they'll be needed again soon (in which case keeping them is fine, but worth a deliberate decision rather than default accumulation).
- Organize utilities by domain (data generation, formatting, retry/wait logic) rather than dumping everything into one large `utils.ts` file — this mirrors the same organizational discipline from [Modules & Project Structure](../03-typescript-playwright/modules-and-project-structure.md).
- When multiple engineers independently reimplement similar logic (two slightly different "generate a unique email" functions appearing in different files), this is a signal the utility library isn't discoverable enough — consider whether documentation or a README index (similar to this repo's own folder READMEs) would help engineers find existing utilities before writing new ones.

## Common Pitfalls

- Extracting a utility function after its first use, before there's any real evidence it'll be needed again — premature abstraction that adds indirection without a proven reuse benefit.
- Letting a utility file grow into an unorganized dumping ground of unrelated functions, making it hard to find what already exists (leading to duplicate reimplementation, ironically the exact problem utilities are meant to solve).
- Writing utilities that are secretly coupled to one specific page or test scenario despite being labeled as generic — this creates false reusability that breaks the moment it's actually reused in a different context.
- Not periodically cleaning up utilities that turned out to only ever be used once — accumulated, never-actually-reused "utilities" add cognitive overhead without real benefit.

## Interview Notes

- Be ready to explain your criteria for deciding whether a piece of logic deserves to be extracted into a shared utility versus staying local to one test or page object.
- Understand the risk of premature abstraction — extracting too early, before genuine reuse is proven — as a real, common mistake, not just "duplication is always bad."
- Be able to describe how you'd organize a growing utility library so it stays discoverable and doesn't become an unstructured dumping ground.

## References

- [Martin Fowler — Refactoring: Extract Function](https://refactoring.com/catalog/extractFunction.html)