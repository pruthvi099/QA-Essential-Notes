# Retries & Timeouts (Python & TypeScript)

## What It Is

Timeouts define how long Playwright waits for a condition (an element to be actionable, a navigation to complete, an assertion to pass) before failing. Retries define whether a *failed test* automatically re-runs — a separate mechanism from the auto-waiting/retrying assertions covered in [Auto-Waiting](./auto-waiting.md) and [Assertions](./assertions.md).

There are two distinct concepts often conflated:
- **Action/assertion retrying** — built into auto-waiting and `expect()`, happens *within* a single test run.
- **Test-level retries** — re-running an entire failed test from scratch, a separate configuration layer for handling genuine flakiness.

## Why It Matters

- Misconfigured timeouts cause two opposite problems: too short causes false failures on legitimately slower operations; too long makes genuinely broken tests take forever to fail, slowing feedback.
- Test-level retries are a pragmatic tool for tolerating occasional infrastructure flakiness (a slow network blip) — but overusing them to mask genuinely broken or badly written tests is one of the most common ways teams quietly erode trust in their suite's signal.
- Interviewers ask about retry strategy specifically to see whether a candidate treats retries as a deliberate, bounded tool or as a blanket flakiness cover-up — the distinction matters a lot for how healthy a team's test suite stays over time.

## How It Works

**Timeout hierarchy (from broadest to narrowest):**
1. Global test timeout — max time for an entire test to complete
2. Action timeout — max time for a single action (click, fill) to become actionable
3. Assertion timeout — max time an `expect()` will retry before failing
4. Navigation timeout — max time for `goto()`/page loads to complete

Each level can be configured globally (config file) or overridden per-call for a specific slower-than-usual operation.

**Test-level retries:**

**Python (pytest, via `pytest-rerunfailures`):**
```bash
pip install pytest-rerunfailures
pytest --reruns 2 --reruns-delay 1
```

**TypeScript (built into Playwright Test):**
```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,   // retry only in CI, not locally
});
```

## Example

**Python — configuring timeouts and a targeted override:**
```python
# pytest.ini
[pytest]
timeout = 30          # global test timeout (seconds)
```

```python
from playwright.sync_api import Page, expect

def test_slow_report_generation(page: Page):
    page.goto("https://example.com/reports")
    page.get_by_role("button", name="Generate Report").click()

    # Override: this specific operation is known to be slow,
    # rather than raising the global timeout for every test
    expect(page.get_by_test_id("report-ready")).to_be_visible(timeout=60000)
```

**TypeScript — equivalent configuration and override:**
```typescript
// playwright.config.ts
export default defineConfig({
  timeout: 30000,        // global test timeout (ms)
  retries: 2,
});
```

```typescript
import { test, expect } from '@playwright/test';

test('slow report generation', async ({ page }) => {
  await page.goto('https://example.com/reports');
  await page.getByRole('button', { name: 'Generate Report' }).click();

  // Override for a known-slow operation
  await expect(page.getByTestId('report-ready')).toBeVisible({ timeout: 60000 });
});
```

**Distinguishing retry types — a test that legitimately needs a retry vs. one that's actually broken:**
```text
Scenario A: Test occasionally fails due to a shared CI runner's network
            latency spike, unrelated to the app or test logic.
            → Reasonable candidate for test-level retry.

Scenario B: Test fails because it doesn't wait for a specific element
            correctly and relies on timing luck.
            → NOT a retry candidate — this needs to be fixed (proper
            locator/assertion), since retrying just hides a real bug
            in the test itself.
```

## Production Considerations

- Enable test-level retries only in CI, not locally — locally, a flaky test should fail loudly and immediately so it gets investigated, not silently pass on the second attempt.
- Track *which* tests actually need retries to pass (most CI reporters flag this) — a test that consistently needs a retry to pass is a signal of a real bug in the test (or the app), not evidence the retry mechanism is working as intended.
- Set global timeouts based on your application's realistic performance profile, not arbitrary defaults — an app with genuinely slow report-generation features needs different defaults than a fast, mostly-static site.

## Common Pitfalls

- Using test-level retries as a blanket fix for flaky tests instead of diagnosing and fixing the underlying cause (usually a missing/incorrect wait, or a genuine race condition in the app).
- Setting retries locally as well as in CI, hiding real flakiness from the person who just wrote the test and is best positioned to fix it immediately.
- Confusing action/assertion retrying (automatic, built into every action/assertion) with test-level retries (a separate, explicit re-run-the-whole-test mechanism) — these operate at completely different levels and solve different problems.
- Setting one global timeout value far higher than needed "to be safe," which makes every genuinely broken test take needlessly long to fail and slows the whole suite's feedback loop.

## Interview Notes

- Be ready to explain the difference between action/assertion retrying (automatic, per-action) and test-level retries (explicit, re-runs the whole test) — a very commonly confused pair.
- Understand why retries should generally be CI-only, not local — and be able to justify it in terms of developer feedback speed.
- Be able to describe how you'd investigate whether a test needing retries indicates a real bug versus genuine environmental flakiness — this shows retry usage as a diagnostic signal, not just a pass-rate booster.

## References

- [Playwright — Test Retries (Node.js)](https://playwright.dev/docs/test-retries)
- [Playwright — Timeouts (Python)](https://playwright.dev/python/docs/test-timeouts)
- [pytest-rerunfailures](https://github.com/pytest-dev/pytest-rerunfailures)