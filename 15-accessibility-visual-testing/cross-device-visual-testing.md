# Cross-Device Visual Testing

## What It Is

Cross-device visual testing extends [Visual Regression Testing](../03-typescript-playwright/visual-regression-testing.md) across multiple viewports and breakpoints, not just a single default screen size — verifying that a responsive layout renders correctly at mobile, tablet, and desktop widths, and that visual regressions specific to one breakpoint don't slip through when only one viewport is tested.

## Why It Matters

- A visual regression can be viewport-specific — a layout that looks perfect at desktop width but breaks at a mobile breakpoint (overlapping elements, text overflow, a hidden navigation menu) is invisible to a visual test suite that only captures one screen size.
- This connects directly to [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md)'s support-matrix thinking — the same "define which combinations actually matter based on real usage" principle applies here, since testing every conceivable viewport size is as impractical as testing every browser/OS combination.
- Responsive design bugs are common and often subtle — a CSS breakpoint that shifts by a few pixels during a refactor can silently break a layout at one specific width while looking fine at the widths a developer happened to check manually.

## How It Works

**The core technique:** parameterize the same visual test across multiple viewport sizes, capturing a separate baseline/snapshot per breakpoint — extending the parameterization pattern from [Config-Driven Test Parameterization](../03-typescript-playwright/typescript-config-driven-test-parameterization.md) specifically to visual testing.

**Choosing which viewports to test:** mirror the same risk-based, usage-data-driven approach from [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md) — common breakpoints (mobile ~375px, tablet ~768px, desktop ~1280px+) as a starting baseline, refined against actual analytics on real visitor viewport widths rather than arbitrary round numbers.

**A key distinction:** cross-device visual testing (this note) captures how a *responsive web layout* renders at different viewport widths in a desktop/mobile browser — this is different from mobile-specific visual testing on real native app devices, which (per current tooling limitations discussed in the broader visual testing ecosystem) is a separate, less mature problem space that web-focused visual tools like Playwright/Percy/Chromatic don't address for native iOS/Android apps.

## Example

**Parameterized visual regression tests across multiple viewports, using Playwright's built-in device emulation:**
```typescript
import { test, expect, devices } from '@playwright/test';

const VIEWPORTS = [
  { name: 'mobile', ...devices['iPhone 13'] },
  { name: 'tablet', viewport: { width: 768, height: 1024 } },
  { name: 'desktop', viewport: { width: 1280, height: 800 } },
];

for (const viewport of VIEWPORTS) {
  test.describe(`Checkout page - ${viewport.name}`, () => {
    test.use({ viewport: viewport.viewport });

    test(`visual regression at ${viewport.name} width`, async ({ page }) => {
      await page.goto('/checkout');
      await expect(page).toHaveScreenshot(`checkout-${viewport.name}.png`);
    });
  });
}
```

**A realistic finding this approach catches that single-viewport testing would miss:**
```text
Bug: A recent CSS refactor changed the checkout page's grid layout.
At desktop width (1280px), it looks perfect — no visible change.
At mobile width (375px), the order summary sidebar now overlaps
the payment form, since the refactor changed a breakpoint threshold
from 768px to 780px, and a previously-mobile-optimized layout rule
no longer applies at exactly 375px.

A visual test suite capturing ONLY desktop width would show this
PR as passing — the regression is entirely invisible outside the
mobile viewport range.
```

**Configuring project-level viewport coverage in `playwright.config.ts`, applying the multi-project pattern from earlier notes specifically to visual testing:**
```typescript
export default defineConfig({
  projects: [
    { name: 'visual-mobile', use: { viewport: { width: 375, height: 812 } } },
    { name: 'visual-tablet', use: { viewport: { width: 768, height: 1024 } } },
    { name: 'visual-desktop', use: { viewport: { width: 1280, height: 800 } } },
  ],
});
```

## Production Considerations

- Multiply cost/runtime awareness by viewport count — visual testing already has a storage and review-time cost (see [Visual Testing Tools Comparison](./visual-testing-tools-comparison.md)); testing at 3 viewports triples the snapshot volume for the same set of pages, which matters directly for free-tier limits on paid platforms.
- Prioritize cross-device coverage for pages with genuinely complex responsive behavior (multi-column layouts, collapsible navigation) over simple, largely static content pages that are less likely to have viewport-specific regressions.
- Base viewport choices on real analytics of actual visitor screen widths, the same principle as [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md) — testing a viewport width that reflects a tiny fraction of real users is a poor use of the added snapshot/review cost.

## Common Pitfalls

- Testing visual regression at only one default viewport (usually desktop, matching a developer's own monitor), missing regressions that only manifest at other breakpoints.
- Testing an excessive number of arbitrary viewport widths without justification, multiplying snapshot count and review burden without proportional coverage value — three to four well-chosen breakpoints usually covers the meaningful range far better than a dozen arbitrary ones.
- Confusing cross-device visual testing (responsive web layouts at different viewport widths) with actual native mobile app visual testing on real devices — these are genuinely different problems requiring different tooling, and conflating them leads to a false sense of mobile coverage.
- Not updating viewport choices as real user device/screen-size distribution shifts over time — a viewport set chosen years ago may no longer reflect current, actual visitor patterns.

## Interview Notes

- Be ready to explain why single-viewport visual testing misses a real, common category of bugs, with a concrete example (an overlap or layout break specific to one breakpoint).
- Understand how to parameterize a visual test across multiple viewports using Playwright's device emulation or project configuration.
- Be able to describe how you'd choose which viewports to prioritize, connecting back to the same risk-based, analytics-driven reasoning from [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md).

## References

- [Playwright — Emulation (Node.js)](https://playwright.dev/docs/emulation)
- [Playwright — Visual Comparisons (Node.js)](https://playwright.dev/docs/test-snapshots)