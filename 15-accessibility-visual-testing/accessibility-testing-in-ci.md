# Accessibility Testing in CI

## What It Is

This note covers gating CI builds on accessibility violations — deciding what severity level blocks a merge versus what only warns, applying the same quality-gate calibration principle from [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md) specifically to axe-core scan results (see [Automated Accessibility Scanning](./automated-accessibility-scanning.md)).

## Why It Matters

- Automated accessibility scanning is only as valuable as its actual enforcement — a scan that runs and reports but never blocks anything allows accessibility regressions to accumulate indefinitely, the same "gate that exists but isn't enforced" risk covered generally in [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md).
- Blocking on every single violation regardless of severity risks the same bypass-training problem discussed for flaky tests and quality gates generally — a team that gets blocked on minor findings routinely will eventually find ways around the gate entirely, including for genuinely critical findings.
- This is a practical, everyday integration decision — an SDET setting up accessibility CI checks needs a specific, calibrated policy, not just "run axe-core and see what happens."

## How It Works

**A typical severity-based gating policy, applying axe-core's impact levels (see [Automated Accessibility Scanning](./automated-accessibility-scanning.md)):**
- **Critical/Serious** — block the merge; these represent significant, confirmed barriers to access.
- **Moderate** — warn/flag for review, but don't block; tracked and addressed on a reasonable timeline.
- **Minor** — logged for visibility/trend tracking, not individually actionable per-PR.

**Where in the pipeline this runs:** accessibility scans on core/critical pages are reasonable to run on every PR (they're fast — axe-core scans a single page in well under a second), unlike full performance test suites; a broader, site-wide scan across every page/route is a better fit for a scheduled, less frequent run, mirroring the smoke/full-regression split from [Smoke vs. Regression in CI](../07-ci-cd/smoke-vs-regression-in-ci.md).

## Example

**A CI step gating on severity, integrated into a Playwright-based pipeline:**
```yaml
# .github/workflows/a11y-check.yml
name: Accessibility Check
on:
  pull_request:
    branches: [main]

jobs:
  a11y_scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Run accessibility scan on critical pages
        run: npx playwright test tests/a11y/critical-pages.spec.ts
```

```typescript
// tests/a11y/critical-pages.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

const CRITICAL_PAGES = ['/login', '/checkout', '/cart', '/'];

for (const path of CRITICAL_PAGES) {
  test(`${path} has no critical/serious a11y violations`, async ({ page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page }).analyze();

    const blocking = results.violations.filter(
      v => v.impact === 'critical' || v.impact === 'serious'
    );
    const nonBlocking = results.violations.filter(
      v => v.impact === 'moderate' || v.impact === 'minor'
    );

    // Non-blocking findings are still surfaced, just don't fail the build
    if (nonBlocking.length > 0) {
      console.warn(`${path}: ${nonBlocking.length} moderate/minor findings — tracked, not blocking`);
    }

    // Only critical/serious findings actually fail the CI check
    expect(blocking, JSON.stringify(blocking, null, 2)).toEqual([]);
  });
}
```

**A broader, scheduled site-wide scan — not PR-blocking, mirroring the nightly full-regression pattern:**
```yaml
# .github/workflows/a11y-full-sweep.yml
name: Full Accessibility Sweep
on:
  schedule:
    - cron: '0 4 * * 1'   # weekly, not on every PR — broader scope, slower

jobs:
  full_sweep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright test tests/a11y/full-site-sweep.spec.ts
      - name: Upload full report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: a11y-full-report
          path: a11y-report/
```

## Production Considerations

- Calibrate the severity threshold with input from accessibility-knowledgeable stakeholders, not purely by SDET judgment alone — what counts as "blocking" versus "tracked" is partly a product/legal risk decision (see [WCAG Conformance Levels](./wcag-conformance-levels.md) for the compliance-target context this decision sits within), not purely a technical one.
- Track moderate/minor findings over time even though they don't block — an accumulating pile of untracked "non-blocking" findings is exactly the kind of technical debt discussed generally in [Test Framework Maintainability & Technical Debt](../10-test-framework-design/test-framework-maintainability-and-technical-debt.md), just applied to accessibility specifically.
- Scope the PR-blocking scan to genuinely critical, high-traffic pages rather than the entire site — this keeps the fast-feedback check fast, while the full-site sweep (scheduled, not blocking) still provides comprehensive coverage.

## Common Pitfalls

- Blocking every PR on any violation regardless of severity, training the team to treat the accessibility gate as noise and look for ways around it — undermining the gate for exactly the findings that matter most.
- Running an accessibility scan in CI that only logs results without ever actually failing the build on anything — a gate that exists in name only provides no real enforcement.
- Scanning only the homepage or a single arbitrary page instead of the actual critical user paths (login, checkout) where accessibility barriers have the most real impact.
- Not tracking non-blocking (moderate/minor) findings anywhere, letting them accumulate indefinitely with no visibility or accountability for eventually addressing them.

## Interview Notes

- Be ready to describe a severity-based accessibility gating policy — what blocks versus what's tracked — and justify the calibration, mirroring the general quality-gate reasoning from [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md) applied specifically here.
- Understand why scanning critical pages on every PR (fast) versus a full site sweep on a schedule (broader, slower) is the practical staging choice, similar to smoke/regression tiering elsewhere in this repo.
- Be able to explain the real risk of blocking on every single finding regardless of severity — the bypass-training problem — with a concrete example.

## References

- [Deque — axe-core GitHub Action / CI Integration](https://github.com/dequelabs/axe-core-npm)
- [W3C — Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/)