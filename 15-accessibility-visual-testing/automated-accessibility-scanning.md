# Automated Accessibility Scanning

## What It Is

This note goes deeper into axe-core specifically — the open-source rules engine underlying most professional accessibility scanning tools (Deque's axe DevTools, Microsoft Accessibility Insights, and the axe-core integration used in [Manual Accessibility Testing](../01-manual-testing/manual-accessibility-testing.md)) — covering its rule structure, severity/impact levels, and precisely what it can and cannot catch. This expands on the brief CI integration shown earlier into a full understanding of the tool itself.

## Why It Matters

- axe-core is specifically engineered for a **zero false positives** design principle — when it can't definitively determine whether something violates a rule, it reports "incomplete" (needs review) rather than a false violation, which is what makes its output trustworthy enough to gate a CI build on (see [Accessibility Testing in CI](./accessibility-testing-in-ci.md)).
- Understanding axe-core's actual outcome categories (violations, incomplete, passes, inapplicable) — not just "pass/fail" — is necessary for correctly interpreting a scan's results and knowing what still needs human judgment.
- As of axe-core 4.9, the engine implements roughly 90 individual rules covering WCAG 2.0/2.1/2.2 (Levels A, AA, AAA) plus Section 508 — a large, real, actively maintained rule set, not a token gesture toward accessibility.

## How It Works

**axe-core's four possible outcomes per rule, per element:**
- **Violation** — definitively fails a WCAG criterion (e.g., a `color-contrast` or `label` failure).
- **Incomplete** — cannot be automatically determined; flagged for manual review (e.g., an image has alt text, but axe-core can't judge whether that text is actually *meaningful*).
- **Pass** — the rule was checked and the element passed.
- **Inapplicable** — the rule doesn't apply to this page (e.g., no `<video>` elements means the caption rule never fires).

**Impact/severity levels** — Deque assigns each rule a fixed impact level (critical, serious, moderate, minor) baked into the rule's own metadata, not calculated dynamically per violation:
- **Critical** — completely prevents access to content/functionality (e.g., a form with no way for a keyboard user to submit it).
- **Serious** — causes significant difficulty (e.g., insufficient color contrast).
- **Moderate** — causes some difficulty/confusion (e.g., a heading hierarchy that skips a level).
- **Minor** — small but measurable impact (e.g., a missing landmark role on secondary navigation).

**What axe-core reliably catches:** color contrast (including against background images via canvas sampling), missing/incorrect ARIA attributes and roles, form fields without programmatic labels, and dozens of other structural/semantic checks.

**What it cannot determine automatically** (and correctly reports as "incomplete" rather than guessing): whether alt text is actually *meaningful* (versus just present), whether a heading's wording makes logical sense, whether a genuinely usable keyboard flow exists end-to-end — these still require the manual checks from [Manual Accessibility Testing](../01-manual-testing/manual-accessibility-testing.md).

## Example

Running an axe-core scan and interpreting the structured result, distinguishing what's automatically actionable from what needs human follow-up:

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('login page has no automatically detectable a11y violations', async ({ page }) => {
  await page.goto('/login');

  const results = await new AxeBuilder({ page }).analyze();

  // Log the full breakdown, not just violation count — "incomplete"
  // items are just as worth reviewing as confirmed violations
  console.log(`Violations: ${results.violations.length}`);
  console.log(`Needs review (incomplete): ${results.incomplete.length}`);
  console.log(`Passed checks: ${results.passes.length}`);

  // Fail the test on any violation — but incomplete items are
  // surfaced for review, not silently ignored
  expect(results.violations).toEqual([]);
});
```

A sample violation object, showing the structure that makes results genuinely actionable rather than a vague score:
```json
{
  "id": "color-contrast",
  "impact": "serious",
  "description": "Ensures the contrast between foreground and background colors meets WCAG 2 AA contrast ratio thresholds",
  "helpUrl": "https://dequeuniversity.com/rules/axe/4.9/color-contrast",
  "nodes": [
    {
      "html": "<button class=\"btn-secondary\">Cancel</button>",
      "target": [".checkout-form button.btn-secondary"]
    }
  ]
}
```

## Production Considerations

- Treat axe-core's "incomplete" results as a genuine action item, not a category to ignore — these are specifically the cases where the tool has determined it *cannot* safely judge pass/fail, which is exactly where manual review (see [Manual Accessibility Testing](../01-manual-testing/manual-accessibility-testing.md)) adds real value beyond automation.
- Different accessibility tools built on different scoring philosophies aren't directly comparable — a Lighthouse accessibility score (a single weighted 0–100 number) and a raw axe-core violation count answer different questions, and comparing scores across tools built on different engines is close to meaningless.
- No automated tool — axe-core included — should be read as a conformance claim; automation reliably catches only a portion of real accessibility issues, and a page with zero detected axe-core violations is not the same as a genuinely accessible page.

## Common Pitfalls

- Treating a clean axe-core scan (zero violations) as proof the page is fully accessible — automated scanning structurally cannot verify things like meaningful alt text wording or logical screen-reader flow.
- Ignoring "incomplete" results because they don't count as hard violations — these are deliberately flagged for a reason and often point to genuinely uncertain areas worth a human look.
- Comparing accessibility scores across different tools (Lighthouse vs. axe vs. WAVE) as if they measure the same thing — each has a different scoring philosophy, and cross-tool comparison isn't meaningful.
- Not knowing that impact levels are fixed per-rule metadata (not computed dynamically) — assuming impact reflects the *specific* page's context can lead to over- or under-prioritizing a given finding relative to its real-world effect on that particular page.

## Interview Notes

- Be ready to explain axe-core's four outcome categories (violation, incomplete, pass, inapplicable) precisely, especially the distinction between "violation" and "incomplete."
- Understand the four impact levels (critical, serious, moderate, minor) with a concrete example of each, and know that these are Deque's own severity assignments, not part of the WCAG standard itself.
- Be able to explain why a clean automated scan doesn't mean a page is accessible — automation catches a meaningful but partial subset of real issues, a specific, well-documented limitation worth citing precisely rather than vaguely.

## References

- [axe-core — GitHub](https://github.com/dequelabs/axe-core)
- [Deque University — axe-core Rule Descriptions](https://dequeuniversity.com/rules/axe/)