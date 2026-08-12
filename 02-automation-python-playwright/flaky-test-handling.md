# Flaky Test Handling (Python & TypeScript)

## What It Is

A flaky test is one that produces different results (pass/fail) across runs without any change to the code under test — the same test, same code, sometimes passes and sometimes fails. Flaky test handling is the systematic practice of identifying, diagnosing, and fixing (or, as a last resort, quarantining) these tests, since flakiness — left unaddressed — destroys trust in an entire automation suite.

## Why It Matters

- Once a team stops trusting a suite's results ("oh, that test is just flaky, ignore it"), the suite stops functioning as a real quality gate — failures get dismissed reflexively, including genuine regressions, which defeats the entire purpose of automation.
- Flaky test triage and root-cause fixing is one of the most common ongoing responsibilities of an SDET on a mature team — this is less about writing new tests and more about maintaining the health of existing ones, a distinction interviewers specifically probe for.
- Nearly every concept covered earlier in this folder (auto-waiting, web-first assertions, test isolation, parallel-safety) is, from a different angle, a tool for *preventing* flakiness — this note ties them together into a diagnostic framework.

## How It Works

**Common root causes of flakiness, roughly in order of frequency:**

1. **Missing/incorrect waits** — asserting before the UI has actually finished updating (see [Auto-Waiting](./auto-waiting.md), [Assertions](./assertions.md))
2. **Test interdependence** — a test relies on state left behind by another test, and passes/fails depending on execution order (see [Browser Contexts & Test Isolation](./browser-contexts-and-test-isolation.md))
3. **Shared external state under parallelism** — multiple workers racing on the same test account/data (see [Parallel Execution](./parallel-execution.md))
4. **Timing-sensitive application behavior** — animations, debounced inputs, or race conditions in the app itself, not just the test
5. **Environment instability** — network flakiness, resource-constrained CI runners, third-party service dependencies

**Diagnostic approach:**
1. **Reproduce** — run the failing test in isolation, repeatedly, to see if it fails consistently alone (rules out interdependence) or intermittently even alone (points to timing/environment).
2. **Inspect artifacts** — use the trace viewer (see [Tracing, Screenshots & Videos](./tracing-screenshots-videos.md)) from an actual failure to see exactly what state the page was in at the moment of failure.
3. **Isolate the cause** — check for missing waits first (the most common cause), then test order dependence, then shared data/parallelism issues.
4. **Fix at the root** — add a proper wait/assertion, isolate test data, or fix an actual application race condition — never just add a `sleep()` or blanket retry as the final fix.

## Example

**Diagnosing a real flaky test — before and after:**

```python
# FLAKY: clicking "Save" triggers an async save; this asserts
# immediately, sometimes before the save has actually completed
def test_save_profile_flaky(page):
    page.goto("/profile")
    page.get_by_label("Name").fill("New Name")
    page.get_by_role("button", name="Save").click()
    assert page.get_by_text("Saved successfully").is_visible()  # races the save
```

```python
# FIXED: using a web-first assertion, which retries until the
# confirmation appears, instead of checking once immediately
from playwright.sync_api import expect

def test_save_profile_fixed(page):
    page.goto("/profile")
    page.get_by_label("Name").fill("New Name")
    page.get_by_role("button", name="Save").click()
    expect(page.get_by_text("Saved successfully")).to_be_visible()  # retries
```

**A test-interdependence example — before and after:**
```python
# FLAKY: assumes a specific cart state left behind by a PREVIOUS test —
# passes in isolation, fails when run after/alongside other tests
def test_cart_has_two_items_flaky(page):
    page.goto("/cart")
    assert page.get_by_test_id("cart-count").inner_text() == "2"
```

```python
# FIXED: creates its own isolated, known starting state instead of
# depending on what another test happened to leave behind
def test_cart_has_two_items_fixed(page):
    page.goto("/products/101")
    page.get_by_role("button", name="Add to Cart").click()
    page.goto("/products/102")
    page.get_by_role("button", name="Add to Cart").click()

    page.goto("/cart")
    assert page.get_by_test_id("cart-count").inner_text() == "2"
```

**TypeScript — the same fix pattern:**
```typescript
// FIXED: web-first assertion instead of a one-time check
test('save profile', async ({ page }) => {
  await page.goto('/profile');
  await page.getByLabel('Name').fill('New Name');
  await page.getByRole('button', { name: 'Save' }).click();
  await expect(page.getByText('Saved successfully')).toBeVisible();
});
```

## Production Considerations

- Track flaky tests explicitly (most CI reporters/dashboards support flagging a test as intermittently failing) — a test that "sometimes needs a retry to pass" should be treated as a known issue with an owner and a fix timeline, not silently tolerated indefinitely.
- Quarantining a flaky test (excluding it from the blocking suite while it's investigated) is a reasonable *temporary* measure, but should have a tracked expiry — permanently quarantined tests that never get fixed are effectively dead code providing no coverage.
- New flaky tests are far cheaper to fix immediately (the author has full context) than months later, once the original author has moved on and no longer remembers the test's intent — flagging and fixing flakiness quickly should be a team norm, not deferred work.

## Common Pitfalls

- Reflexively adding a `sleep()` or blanket test-level retry the moment a test is flaky, without ever diagnosing the actual root cause — this hides the symptom while leaving the underlying bug (in the test or the app) unresolved.
- Ignoring a known-flaky test indefinitely ("that one's just flaky") — this normalizes distrust of the suite and often masks a genuine, intermittent application bug worth investigating in its own right.
- Diagnosing flakiness by staring at code instead of using the trace viewer/artifacts from an actual failure — this wastes time guessing when the evidence needed is often already captured (see [Tracing, Screenshots & Videos](./tracing-screenshots-videos.md)).
- Not distinguishing test-caused flakiness (missing waits, shared state) from genuine application bugs (a real race condition in production code) — the latter is a legitimate defect that deserves its own bug report, not just a test-side fix.

## Interview Notes

- Be ready to walk through a systematic diagnostic process for a flaky test — reproduce, inspect artifacts, isolate cause, fix at the root — rather than jumping straight to "add a retry," which interviewers specifically probe for.
- Be able to categorize common flakiness root causes (missing waits, test interdependence, shared state under parallelism, environment instability) and map each to the concept/tool that prevents it.
- Understand why an occasionally-flaky test is sometimes actually surfacing a real, intermittent application bug — not always a test-code problem — and be able to describe how you'd investigate which it is.

## References

- [Playwright — Best Practices (Node.js)](https://playwright.dev/docs/best-practices)
- [Google Testing Blog — Flaky Tests](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)