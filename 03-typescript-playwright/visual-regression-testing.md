# Visual Regression Testing

## What It Is

Visual regression testing captures screenshots of a page or component and compares them pixel-by-pixel (or via perceptual diffing) against a stored baseline image, flagging any visual difference — even ones that don't break any functional assertion. Playwright's built-in `toHaveScreenshot()` matcher provides this natively, without needing a separate visual testing tool for basic use cases.

## Why It Matters

- Functional tests (does the button work, does the form submit) can pass while a page visually breaks — a CSS regression, a misaligned layout, an unintended style change — none of which a functional assertion would ever catch, but a visual diff would.
- This complements [Manual Accessibility Testing](../01-manual-testing/manual-accessibility-testing.md) and [Usability Testing](../01-manual-testing/usability-testing.md) by automating the part of "does it look right" that's stable and repeatable enough to check on every run, while human judgment stays reserved for genuinely subjective visual/UX calls.
- Interviewers increasingly ask about visual testing as design systems and component libraries have made visual regressions a common, real production issue — knowing the tool and its limitations (flakiness from fonts/animations, cross-platform rendering differences) shows current, practical framework knowledge.

## How It Works

**First run** — `toHaveScreenshot()` captures and saves a baseline image if none exists yet.

**Subsequent runs** — the current screenshot is compared against the stored baseline; if the difference exceeds a configurable threshold, the test fails and Playwright generates a diff image highlighting exactly what changed.

**Key configuration:**
- `maxDiffPixels` / `maxDiffPixelRatio` — tolerance for how many pixels can differ before failing, useful for tolerating minor anti-aliasing differences across environments.
- `threshold` — per-pixel color difference sensitivity.
- Masking (`mask: [locator]`) — exclude genuinely dynamic regions (a live clock, an ad banner) from comparison entirely, rather than letting them cause false failures every run.

## Example

Basic full-page and component-level visual regression tests:

```typescript
import { test, expect } from '@playwright/test';

test('homepage visual regression', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png');
});

test('checkout button visual regression', async ({ page }) => {
  await page.goto('/checkout');
  const button = page.getByRole('button', { name: 'Place Order' });
  await expect(button).toHaveScreenshot('checkout-button.png');
});
```

Handling a genuinely dynamic region (a "last updated" timestamp) that would otherwise cause a false failure on every run:

```typescript
test('dashboard visual regression, excluding dynamic timestamp', async ({ page }) => {
  await page.goto('/dashboard');

  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.getByTestId('last-updated-timestamp')],
    maxDiffPixelRatio: 0.01,   // small tolerance for minor rendering differences
  });
});
```

Updating baselines intentionally after a deliberate design change:
```bash
npx playwright test --update-snapshots
```

## Production Considerations

- Run visual regression tests on a single, consistent browser/OS combination in CI (rendering differs subtly across platforms) — comparing screenshots generated on a developer's local machine against a baseline generated in CI is a common, avoidable source of false failures.
- Mask or exclude genuinely dynamic content (timestamps, ads, animations, randomized data) explicitly — without this, visual tests fail on essentially every run for reasons unrelated to real regressions, quickly eroding trust in the suite the same way general flakiness does (see [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md)).
- Treat baseline image updates as a reviewed, deliberate action (part of a PR, visible in the diff) — not something updated casually, since an uncritically approved new baseline can accidentally "bless" a real visual regression as the new expected state.

## Common Pitfalls

- Running visual tests across different OS/browser environments than where the baseline was captured, causing font-rendering and anti-aliasing differences to be flagged as false regressions.
- Not masking dynamic content, leading to visual tests that fail almost every run regardless of whether anything meaningful actually changed — teams often abandon visual testing entirely because of this, when masking would have solved it.
- Blindly running `--update-snapshots` to "fix" failing visual tests without reviewing what actually changed — this can silently approve a real, unintended visual regression as the new baseline.
- Applying visual regression testing to every page/component indiscriminately instead of prioritizing high-visibility, high-risk UI (checkout, core navigation, design-system components) — this adds maintenance overhead disproportionate to the value for low-risk pages.

## Interview Notes

- Be ready to explain what visual regression testing catches that functional assertions don't, with a concrete example (a CSS layout break with no functional impact).
- Understand why cross-environment rendering differences are the primary practical challenge with visual testing, and how masking/tolerance settings address (some of) it.
- Be able to describe how you'd decide which pages/components warrant visual regression coverage, rather than applying it universally — this ties back to risk-based prioritization (see [Risk-Based Testing](../00-start-here/risk-based-testing.md)).

## References

- [Playwright — Visual Comparisons (Node.js)](https://playwright.dev/docs/test-snapshots)