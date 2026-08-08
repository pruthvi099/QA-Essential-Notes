# Manual Regression Testing

## What It Is

Manual regression testing is the practice of re-executing existing test cases by hand to confirm that a code change hasn't broken previously working functionality. It's distinct from testing the new change itself — regression testing verifies everything *around* the change still works, not the change in isolation.

This note covers running regression manually — typically before automation coverage exists for an area, or as a supplement to automated regression for things automation doesn't reliably catch (visual/UX regressions, exploratory-style checks).

## Why It Matters

- Regression is one of the most repetitive, time-consuming manual testing activities — which is exactly why it's usually the first thing teams automate. Understanding how to run it well manually is what tells you *what* to automate first and how to scope that automation.
- In early-stage projects or newly changed areas without automation yet, manual regression is often the only safety net against a change breaking unrelated functionality — skipping it under time pressure is a common, costly mistake.
- As an SDET, you're frequently the person deciding which manual regression suite items get automated next — this requires understanding regression suite structure and pain points firsthand, not just writing new automation.

## How It Works

**Structuring a manual regression suite:**
- Organize by feature area, not by when the test case was written — a tester should be able to find "all checkout regression tests" together.
- Tag/prioritize by risk (see [Risk-Based Testing](../00-start-here/risk-based-testing.md)) — critical-path tests (P1) run every cycle; lower-priority ones may run less frequently or be first in line for automation.
- Keep it current — remove test cases for deprecated features, add new ones as features stabilize enough to be "regression-worthy" (see [Manual Testing vs. Automation Testing](../00-start-here/manual-testing-vs-automation-testing.md) for the automate-or-not decision).

**Full vs. partial (smoke) regression:**
- **Full regression** — every test case in the suite; slow, used before major releases.
- **Partial/targeted regression** — only test cases in areas related to the actual change (impact analysis); faster, used for smaller, frequent releases.

**Impact analysis** — before running regression, identify which areas of the app the change could plausibly affect (shared components, related data models, adjacent flows) so partial regression is scoped correctly rather than guessed.

## Example

A regression suite excerpt with priority tagging, and a targeted (impact-analysis-based) run for a small change:

```text
Regression Suite — Checkout Module (excerpt)

TC_CHECKOUT_001 [P1] Verify order completes with valid card
TC_CHECKOUT_002 [P1] Verify order fails gracefully with declined card
TC_CHECKOUT_003 [P1] Verify discount code applies correct amount
TC_CHECKOUT_004 [P2] Verify order summary shows correct item list
TC_CHECKOUT_005 [P2] Verify guest checkout (no account) completes
TC_CHECKOUT_006 [P3] Verify order confirmation email content is correct
TC_CHECKOUT_007 [P3] Verify "Continue Shopping" link returns to catalog
```

```text
Change: Updated discount code validation logic (backend only, no UI change)

Impact Analysis:
  Directly affected: Discount code entry, application, and total
  calculation at checkout
  Not affected: Payment processing, order confirmation, guest checkout
  (no shared code path with discount logic)

Targeted Regression Run (not full suite):
  TC_CHECKOUT_001 [P1] — still relevant, discount interacts with total
  TC_CHECKOUT_003 [P1] — directly tests the changed logic
  TC_CHECKOUT_004 [P2] — order summary reflects discount, worth checking
  Skipped: TC_CHECKOUT_002, 005, 006, 007 — no code path overlap with
  the change, full regression reserved for pre-release instead
```

This targeted approach turns a 7-test full run into a focused 3-test run for this specific change — full regression (all 7) still runs before the actual release, but doesn't need to run after every small, scoped change.

## Production Considerations

- Manual regression suites should be treated as a prioritized automation backlog — the test cases run most frequently, with the most rigid expected results, are usually the best early automation candidates (see [QA Engineer vs. SDET](../00-start-here/qa-engineer-vs-sdet.md) on framework-building as an SDET responsibility).
- Impact analysis quality depends on understanding the codebase's architecture (shared components, data flows) — this is a place where close collaboration with developers meaningfully reduces wasted regression effort.
- As automation coverage grows, manual regression should shrink to cover only what automation doesn't yet reach — a manual regression suite that never shrinks as automation grows signals automation isn't actually replacing repetitive manual work.

## Common Pitfalls

- Running full regression for every small change regardless of actual impact, wasting significant tester time on unrelated areas.
- Letting the regression suite grow indefinitely without pruning outdated or duplicate test cases — an unmaintained suite becomes too slow to run consistently and gets skipped under pressure.
- Skipping regression entirely under release time pressure without a documented risk acceptance — this is how previously-fixed bugs silently reappear in production ("regression" in the literal sense).
- Doing impact analysis carelessly (assuming a change is isolated when it actually touches shared code), causing missed regression coverage in an affected area.

## Interview Notes

- Be ready to explain how you'd scope a regression run for a specific change (impact analysis) rather than defaulting to "run everything."
- Understand why regression testing is typically the first automation priority in a growing project — repetition + stable expected results make it the highest-ROI automation target.
- Be able to describe how you'd decide when a manual regression suite has grown too large/slow and needs pruning or increased automation investment.

## References

- [ISTQB Foundation Level Syllabus — Regression Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)