# Config-Driven Test Parameterization

## What It Is

Test parameterization runs the same test logic against multiple sets of input data or configuration, without duplicating the test itself. In Playwright TypeScript, this is done via `test.describe()` loops over typed data arrays, and via `projects` in `playwright.config.ts` for running the same suite across different browsers, devices, or environments.

## Why It Matters

- Writing a near-identical test for every input variant (each discount code, each user role, each browser) creates massive duplication — a single bug fix or assertion change then needs to be repeated across every copy, which is exactly the maintenance burden parameterization eliminates.
- Config-driven `projects` is how a single suite scales to "run on Chrome, Firefox, Safari, and mobile viewports" without writing four separate copies of every test file — this is one of Playwright Test's most distinctive built-in capabilities compared to older tools.
- Interviewers ask about parameterization to see whether a candidate writes DRY, scalable test code or defaults to copy-pasting variations — a strong, practical signal of framework maturity.

## How It Works

**Data-driven parameterization** — looping over a typed array of test cases and generating one test per entry:
```typescript
const testCases: { code: string; expectedDiscount: number }[] = [
  { code: 'SAVE10', expectedDiscount: 10 },
  { code: 'SAVE20', expectedDiscount: 20 },
  { code: 'EXPIRED', expectedDiscount: 0 },
];

for (const { code, expectedDiscount } of testCases) {
  test(`applies correct discount for code ${code}`, async ({ page }) => {
    // ... test body using `code` and `expectedDiscount`
  });
}
```

**Project-based parameterization** (`playwright.config.ts`) — running the entire suite across multiple browsers/devices:
```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
  ],
});
```

Each project runs the *entire* test suite once per configuration — no test code duplication needed for cross-browser/device coverage (this directly extends [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md) with automated, config-driven execution).

## Example

A typed, data-driven test for discount code validation, combined with project-based cross-browser execution:

```typescript
// test-data/discountCodes.ts
export interface DiscountTestCase {
  code: string;
  orderTotal: number;
  expectedTotal: number;
  description: string;
}

export const discountTestCases: DiscountTestCase[] = [
  { code: 'SAVE10', orderTotal: 1000, expectedTotal: 900, description: 'valid 10% code' },
  { code: 'SAVE20', orderTotal: 1000, expectedTotal: 800, description: 'valid 20% code' },
  { code: 'EXPIRED', orderTotal: 1000, expectedTotal: 1000, description: 'expired code, no discount' },
  { code: 'INVALID123', orderTotal: 1000, expectedTotal: 1000, description: 'nonexistent code' },
];
```

```typescript
// tests/discount.spec.ts
import { test, expect } from '@playwright/test';
import { discountTestCases } from '../test-data/discountCodes';

for (const testCase of discountTestCases) {
  test(`discount: ${testCase.description}`, async ({ page }) => {
    await page.goto('/checkout');
    await page.getByLabel('Discount Code').fill(testCase.code);
    await page.getByRole('button', { name: 'Apply' }).click();

    await expect(page.getByTestId('order-total')).toHaveText(
      `₹${testCase.expectedTotal}`
    );
  });
}
```

Running `npx playwright test` with the multi-project config above executes all four of these generated tests once per browser project (chromium, firefox, webkit, mobile-chrome) — 16 total test runs from one data array and one config file, with zero duplicated test code.

## Production Considerations

- Keep parameterized test data in a separate, typed file (as shown) rather than inline in the test file — this makes it easy to extend with new cases without touching test logic, and keeps the data reviewable on its own.
- Not every test needs to run across every project — use `test.skip()` conditionally, or Playwright's per-project test filtering, to exclude tests that are irrelevant for certain browsers/devices (e.g., a desktop-only drag-and-drop feature skipped on mobile projects).
- Balance parameterization breadth against CI runtime — running every test across every browser and device multiplies total execution time; reserve full cross-project runs for critical paths and consider a lighter single-browser run for less critical tests.

## Common Pitfalls

- Copy-pasting near-identical tests for each input variant instead of looping over a data array — this is the exact duplication parameterization is meant to eliminate, and it multiplies maintenance cost with every new variant.
- Adding every browser/device to `projects` without considering CI time cost — a suite that takes 5 minutes on one browser takes 20+ across four projects; this needs to be a deliberate trade-off, not a default.
- Not typing the parameterized data array — an untyped array of test cases loses the compile-time safety that catches a malformed test case (missing field, wrong type) before it silently produces a confusing test failure instead.
- Using `test.describe()` loops with non-deterministic data (e.g., randomly generated values) that change between runs, making failures hard to reproduce — parameterized data should be fixed and known, not randomized per run.

## Interview Notes

- Be ready to convert a set of near-duplicate tests into a parameterized loop over typed data — a common practical refactoring exercise.
- Understand how `projects` in `playwright.config.ts` provides cross-browser/device coverage without duplicating test files, and be able to explain the CI time trade-off of adding more projects.
- Be able to describe how you'd decide which tests need full cross-browser coverage versus which are fine running on a single browser only — this connects back to risk-based thinking (see [Risk-Based Testing](../00-start-here/risk-based-testing.md)).

## References

- [Playwright — Parameterize Tests (Node.js)](https://playwright.dev/docs/test-parameterize)
- [Playwright — Projects (Node.js)](https://playwright.dev/docs/test-projects)