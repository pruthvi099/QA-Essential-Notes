# Flaky Test Quarantine in CI

## What It Is

This note covers the pipeline-level workflow for handling flaky tests at scale — a structured **quarantine** mechanism that isolates known-flaky tests from blocking the main pipeline while still running and tracking them, distinct from the test-level diagnostic techniques in [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md). Where that note covers *diagnosing and fixing* a single flaky test, this one covers *managing flakiness as an ongoing pipeline-scale problem*.

## Why It Matters

- At scale (hundreds or thousands of tests), some tests will be flaky at any given moment despite best efforts — a pipeline with zero tolerance (any failure blocks everything) becomes unusable if even a small percentage of tests are flaky, while a pipeline that quietly tolerates all failures loses its value as a quality gate entirely.
- A quarantine system lets a team have both: a reliable, blocking pipeline for known-good tests, and continued visibility into flaky tests without them blocking unrelated work — directly connecting to [Quality Gates & Build Failures](./quality-gates-and-build-failures.md)'s "block on flaky checks trains bypass behavior" problem.
- This is genuinely advanced, scale-specific pipeline design that few smaller projects need — but it's a common, practical topic in interviews for roles at companies with large, mature test suites, where flakiness management is an ongoing operational concern, not a one-time fix.

## How It Works

**The quarantine workflow:**
1. A test is identified as flaky (intermittent failures with no code change, or a flagged pattern from CI history).
2. It's tagged as quarantined (`@quarantined` or equivalent) and moved out of the blocking pipeline stage.
3. It continues running in a separate, non-blocking stage — so it's still monitored, and its flakiness (or eventual reliability) is tracked over time.
4. It's actively investigated and fixed — quarantine is meant to be *temporary*, with an expectation of resolution, not a permanent parking lot.
5. Once reliably passing again, it's un-quarantined and moved back into the blocking pipeline.

**What makes this different from just deleting/ignoring a flaky test:** quarantine keeps the test *running and visible* — you still see if it's passing or failing, and can track its reliability trend — it just stops being able to block unrelated merges while under investigation.

## Example

**Tagging and CI structure for quarantine (TypeScript, extending [Test Annotations & Tagging](../03-typescript-playwright/test-annotations-and-tagging.md)):**
```typescript
// A test identified as flaky, with a tracked reason and owner
test('checkout completes with saved payment method @quarantined', async ({ page }) => {
  // JIRA-4821: intermittently fails on CI due to a timing issue with
  // the saved-payment-method dropdown. Quarantined 2026-08-10, owner: @priya
  // ...
});
```

```yaml
# Blocking pipeline stage — quarantined tests EXCLUDED
- name: Run blocking test suite
  run: npx playwright test --grep-invert @quarantined

# Separate, NON-blocking stage — quarantined tests still run and tracked
- name: Run quarantined tests (monitoring only, does not block)
  run: npx playwright test --grep @quarantined
  continue-on-error: true

- name: Report quarantine status
  if: always()
  run: |
    echo "Quarantined test results logged for tracking — see dashboard"
    # In practice, this would push results to a tracking system/dashboard,
    # not just a log line, so quarantine duration and trends are visible
```

A quarantine tracking log — the artifact that keeps quarantine *temporary* rather than a silent, permanent exclusion:
```text
Quarantine Log

Test: checkout completes with saved payment method
Quarantined: 2026-08-10
Reason: Intermittent timeout on payment dropdown (JIRA-4821)
Owner: @priya
Status: Under investigation
Days in quarantine: 10
Escalation threshold: 14 days (flag to team lead if still unresolved)

Test: search returns results within 2s
Quarantined: 2026-07-15
Reason: CI runner performance variance, not a real app issue (JIRA-4790)
Owner: @sam
Status: Fixed — adjusted timeout, re-verified stable for 5 consecutive
        runs, UN-QUARANTINED 2026-08-05
```

## Production Considerations

- Quarantine must have an active tracking mechanism (a dashboard, a log, a labeled ticket) with visible age/duration — without this, quarantine silently becomes a permanent dumping ground for tests nobody ever revisits, which defeats its entire purpose.
- Set an explicit escalation policy (e.g., "flag to team lead if still quarantined after 14 days") — this creates accountability and prevents quarantine from growing unbounded as tests are added but rarely un-quarantined.
- Every quarantined test needs a named owner and a linked investigation ticket, not just a tag — an untracked, unowned quarantined test is functionally identical to a deleted test in terms of actual quality protection, just with extra CI runtime cost.

## Common Pitfalls

- Quarantining a test and never revisiting it — this is the single biggest risk of the whole system, turning quarantine into a slow, silent erosion of real test coverage over time.
- Quarantining tests too readily, without genuine flakiness diagnosis first (see [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md)) — some "flaky" tests are actually consistently failing due to a real bug, and quarantining them without investigation hides a genuine defect rather than managing a genuinely intermittent one.
- Not distinguishing test-caused flakiness from a genuine, intermittent application bug when quarantining — quarantining hides the *symptom* from the blocking pipeline either way, but the appropriate follow-up action differs significantly.
- Having a quarantine mechanism without any escalation or review cadence, so the quarantine list only ever grows and never shrinks.

## Interview Notes

- Be ready to design a quarantine workflow for a large, flaky-at-scale test suite — a common, practical question for roles at companies with mature, large test suites specifically.
- Understand the distinction between this note (managing flakiness at pipeline scale, with tracking and escalation) and [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md) (diagnosing and fixing a single flaky test) — interviewers sometimes ask both, expecting different, complementary answers.
- Be able to explain why quarantine without active tracking/escalation is worse than not having quarantine at all — a sharper, more memorable point than just "quarantine is good practice."

## References

- [Google Testing Blog — Flaky Tests at Google and How We Mitigate Them](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)
- [Playwright — Test Retries (Node.js)](https://playwright.dev/docs/test-retries)