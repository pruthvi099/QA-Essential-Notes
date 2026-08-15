# Test Annotations & Tagging

## What It Is

Playwright Test provides annotations (`test.skip()`, `test.fixme()`, `test.fail()`, `test.slow()`) to mark a test's expected status or behavior, and a tagging system (`@tag` in test titles, or the `tag` option) to selectively run subsets of a suite — e.g., only smoke tests, or excluding known-flaky tests. This extends the smoke/sanity/regression tagging pattern introduced with pytest markers (see [Types of Testing](../00-start-here/types-of-testing.md)) into Playwright Test's own, built-in mechanism.

## Why It Matters

- Not every test should run in every context — a full regression suite is too slow for a pre-commit smoke check, and annotations/tags are how a single codebase serves both needs without maintaining separate test files.
- `test.fixme()` and `test.skip()` with a documented reason are how a team tracks known, temporarily-broken tests without either deleting them (losing the coverage intent) or leaving them failing and polluting CI signal.
- This is a practical, everyday framework-organization skill — interviewers ask about it to see whether a candidate can structure a suite for real CI needs (fast pre-commit checks vs. full nightly runs), not just write individual passing tests.

## How It Works

**Annotations:**
- `test.skip(condition, reason)` — skip a test conditionally (e.g., skip a feature test if it's behind a disabled feature flag)
- `test.fixme()` — mark a test as a known failure to be fixed later; Playwright skips it but reports it distinctly from a passing test
- `test.fail()` — mark a test as *expected* to fail (useful for documenting a known bug with a reproducing test, without breaking the CI build)
- `test.slow()` — triples the timeout for a specific test known to legitimately take longer

**Tagging:**
```typescript
test('applies discount code @smoke @regression', async ({ page }) => { /* ... */ });
```

```bash
npx playwright test --grep @smoke        # run only smoke-tagged tests
npx playwright test --grep-invert @slow  # run everything EXCEPT slow-tagged tests
```

## Example

A realistic mix of annotations and tags in a checkout test file:

```typescript
import { test, expect } from '@playwright/test';

test('completes checkout with valid card @smoke @regression', async ({ page }) => {
  await page.goto('/checkout');
  // ... critical-path test, should always run, even in fast pre-commit checks
});

test('applies discount code correctly @regression', async ({ page }) => {
  // ... broader regression coverage, not part of the fast smoke set
});

test.skip(process.env.FEATURE_GIFT_CARDS !== 'enabled', 'Gift cards feature flag disabled')(
  'redeems a gift card at checkout @regression',
  async ({ page }) => {
    // ... only runs when the feature flag is actually on
  }
);

test.fixme('handles concurrent discount code application race condition', async ({ page }) => {
  // Known bug (see JIRA-4521): tracked here instead of silently
  // dropped or left failing in CI
});

test.fail('checkout total is wrong when currency is JPY (known bug JIRA-4530)', async ({ page }) => {
  await page.goto('/checkout?currency=JPY');
  const total = await page.getByTestId('order-total').innerText();
  expect(total).toBe('¥1000');   // currently fails — expected, tracked as a known bug
});

test.slow();
test('generates a large export report @regression', async ({ page }) => {
  // Legitimately takes longer than the default timeout — tripled here
  // rather than raising the GLOBAL timeout for every test
});
```

```bash
# Fast pre-commit check: only the smoke-tagged test runs
npx playwright test --grep @smoke

# Full nightly run: everything, including regression-tagged tests
npx playwright test --grep @regression
```

## Production Considerations

- Always attach a reason string to `test.skip()`/`test.fixme()` (a ticket reference, a clear explanation) — an unexplained skip is indistinguishable from forgotten dead code six months later, and nobody knows if it's safe to delete or still relevant.
- Standardize a small, consistent tag vocabulary across the team (`@smoke`, `@regression`, `@slow`) rather than letting tags proliferate ad hoc — inconsistent tagging makes `--grep` filtering unreliable across a growing suite.
- Track `test.fixme()` and `test.fail()` counts over time — a growing pile of "known failures" is a signal of accumulating technical debt in either the app or the test suite that needs active attention, not indefinite tolerance.

## Common Pitfalls

- Using `test.skip()` without a reason, or leaving skipped tests in place indefinitely without revisiting them — this silently erodes real coverage while looking like the test still exists.
- Confusing `test.fixme()` (known broken, to be fixed) with `test.fail()` (expected to fail, documenting a known bug intentionally) — they serve different purposes and mixing them up misrepresents the actual state of a test.
- Tagging inconsistently (some tests use `@smoke`, others use `@Smoke` or `#smoke`) — `--grep` matching is exact, so inconsistent casing/format silently excludes tests from filtered runs without any error.
- Overusing `test.slow()` broadly instead of investigating why a test is actually slow — sometimes a "slow" test is masking an unnecessary wait or an inefficient setup step worth fixing at the root.

## Interview Notes

- Be ready to explain the difference between `test.skip()`, `test.fixme()`, and `test.fail()` precisely — a specific, commonly asked Playwright Test question that trips up candidates who've only used `skip`.
- Understand how tagging + `--grep`/`--grep-invert` enables running different subsets of a suite for different CI stages (pre-commit smoke vs. nightly full regression) — connects directly to [Types of Testing](../00-start-here/types-of-testing.md) and CI staging strategy.
- Be able to explain why every skip/fixme needs a documented reason, and what risk an undocumented one creates over time.

## References

- [Playwright — Annotations (Node.js)](https://playwright.dev/docs/test-annotations)
- [Playwright — Tag Tests (Node.js)](https://playwright.dev/docs/test-annotations#tag-tests)