# Combining Visual and Functional Assertions

## What It Is

This note covers the judgment call of when a visual snapshot (see [Visual Regression Testing](../03-typescript-playwright/visual-regression-testing.md)) is the right check for a given scenario versus when a functional assertion (see [Assertions](../02-automation-python-playwright/assertions.md)) is — and how the two complement each other within the same test suite rather than being competing choices for the same verification need.

## Why It Matters

- Visual snapshots and functional assertions catch genuinely different bug classes — a visual snapshot catches a CSS layout break with zero functional impact; a functional assertion catches a business-logic error with zero visual symptom — relying on only one leaves a real, predictable gap.
- Overusing visual snapshots where a functional assertion would be more appropriate creates a maintenance burden without proportional value — every UI tweak requires re-approving a screenshot baseline, even for changes with no actual regression risk.
- This is a practical, everyday test-design decision an SDET makes constantly — being able to articulate the decision criteria clearly (not just "use both sometimes") is what distinguishes deliberate test design from reflexive habit.

## How It Works

**When a visual snapshot is the right tool:**
- Verifying overall layout/design-system consistency (spacing, alignment, color application) where the "correctness" is fundamentally about *appearance*, not a specific extractable value.
- Catching unintended CSS side effects from unrelated code changes (a shared stylesheet change breaking an unrelated page).
- Design-system component libraries (see [Visual Testing Tools Comparison](./visual-testing-tools-comparison.md)'s discussion of Chromatic/Storybook) where the component's visual appearance genuinely *is* the contract being tested.

**When a functional assertion is the better tool:**
- Verifying a specific value is correct (an order total, a displayed username, a status label) — asserting on the exact text via `toHaveText()` is more precise, more maintainable, and gives a clearer failure message than a visual diff would.
- Verifying interactive behavior (a button click produces the correct state change) — a snapshot shows *that* something looks different, not *why*, whereas a functional assertion on the underlying state is directly diagnostic.
- Anything where a snapshot would need to be re-approved for routine, expected content changes (a dynamically changing date, a frequently-updated promotional banner) — visual snapshots are a poor fit for content that's *expected* to vary.

**The synthesis:** many real test suites use both together in the same test — functional assertions for the specific, extractable facts that must be correct, and a visual snapshot as a broader "does the overall page still look right" sweep — each catching what the other structurally cannot.

## Example

**A checkout test combining both appropriately, showing the division of labor concretely:**
```typescript
import { test, expect } from '@playwright/test';

test('checkout page displays correct total and renders correctly', async ({ page }) => {
  await page.goto('/checkout');
  await addItemToCart(page, { price: 1500, qty: 2 });

  // FUNCTIONAL ASSERTION: verifies the EXACT, extractable business
  // value is correct — precise, gives a clear failure message,
  // and doesn't need re-approval if the layout changes but the
  // total calculation logic doesn't
  await expect(page.getByTestId('order-total')).toHaveText('₹3,000');

  // VISUAL SNAPSHOT: verifies the overall page layout/design hasn't
  // broken — catches CSS regressions a functional assertion
  // structurally cannot detect (e.g., the total is CORRECT but
  // visually overlapping another element)
  await expect(page).toHaveScreenshot('checkout-page.png');
});
```

**A counter-example showing visual snapshotting misapplied to something a functional assertion would handle better:**
```typescript
// POOR CHOICE: using a visual snapshot to verify a specific text value
test('order total is correct - BAD APPROACH', async ({ page }) => {
  await page.goto('/checkout');
  await expect(page.locator('.order-total')).toHaveScreenshot('total-value.png');
  // Problems: any font rendering difference, any unrelated nearby
  // layout shift, or even antialiasing variance could fail this test
  // for reasons having NOTHING to do with whether the total is
  // actually correct — and the failure message is a pixel diff,
  // not "expected ₹3,000, got ₹2,850"
});
```

```typescript
// BETTER: a precise functional assertion for the same need
test('order total is correct', async ({ page }) => {
  await page.goto('/checkout');
  await expect(page.locator('.order-total')).toHaveText('₹3,000');
  // Clear, specific, doesn't break on unrelated visual noise,
  // and gives an immediately actionable failure message
});
```

**A scenario where visual snapshotting is genuinely necessary because there's no clean, extractable functional assertion:**
```typescript
test('design system button component renders per spec', async ({ mount }) => {
  const component = await mount(<Button variant="primary">Submit</Button>);

  // There's no single "correct value" to assert on here — the
  // actual thing being verified (correct padding, border-radius,
  // color application per the design system) IS fundamentally visual
  await expect(component).toHaveScreenshot('primary-button.png');
});
```

## Production Considerations

- Default to a functional assertion whenever the thing being verified has a specific, extractable correct value — reserve visual snapshots for genuinely visual concerns (layout, design-system fidelity) where no clean functional assertion exists.
- In a single test, layering a functional assertion (for the specific value) alongside a broader visual snapshot (for overall page health) is a reasonable, common pattern — not redundant, since each catches a genuinely different failure mode.
- When a visual snapshot test starts failing frequently for reasons unrelated to real regressions (see [Visual Regression Testing](../03-typescript-playwright/visual-regression-testing.md)'s masking/dynamic-content cautions), that's a signal the scenario might be better served by a functional assertion instead, or that the visual test needs better masking of dynamic regions.

## Common Pitfalls

- Using a visual snapshot to verify a specific text/numeric value, creating a test that's both less precise (a pixel diff instead of an exact expected value) and more fragile (breaks on unrelated visual noise) than a functional assertion would be.
- Relying only on functional assertions and never adding visual coverage, missing CSS/layout regressions that have no functional symptom a targeted assertion would ever catch.
- Not recognizing when a scenario genuinely has no clean functional assertion available (pure design-system/appearance correctness) and forcing an awkward functional check where a visual snapshot is actually the more natural, correct tool.
- Treating visual and functional assertions as interchangeable rather than complementary, missing the deliberate division-of-labor reasoning that makes combining them valuable.

## Interview Notes

- Be ready to explain the decision criteria for choosing a visual snapshot versus a functional assertion for a given scenario — specificity/extractability of the expected value is the core distinguishing factor.
- Understand why using a visual snapshot for a precise value (like a price) is a worse choice than a functional assertion, with the concrete failure-message and fragility reasoning from the example above.
- Be able to describe a realistic test that combines both appropriately in a single scenario, showing you understand them as complementary rather than competing techniques.

## References

- [Playwright — Test Assertions (Node.js)](https://playwright.dev/docs/test-assertions)
- [Playwright — Visual Comparisons (Node.js)](https://playwright.dev/docs/test-snapshots)