# Tracing, Screenshots & Videos (Python & TypeScript)

## What It Is

Playwright captures rich debugging artifacts for test runs — **traces** (a full timeline of actions, network requests, DOM snapshots, and console logs), **screenshots** (image captures at a point in time or on failure), and **videos** (a recording of the entire test execution). These are what turn a failing CI test from "it failed, good luck" into a fully reproducible, inspectable record — critical since CI failures can't be debugged interactively the way a local failure can.

## Why It Matters

- CI test failures happen on a remote machine you can't attach a debugger to — traces/screenshots/videos are the only way to understand *what actually happened* without being able to watch it live.
- The trace viewer specifically shows a DOM snapshot for every action, meaning you can inspect exactly what the page looked like at each step, not just read a stack trace — this is a significantly higher-fidelity debugging tool than most other test frameworks offer.
- This directly extends [Defect Reporting Best Practices](../01-manual-testing/defect-reporting-best-practices.md) — attaching a trace/screenshot/video to an automated failure is the automated equivalent of a well-written manual bug report's "evidence" section.

## How It Works

**Trace** — records actions, network activity, console logs, and DOM snapshots throughout a test; opened afterward in the interactive **Trace Viewer**, which lets you step through the test action-by-action, inspecting the DOM state and network calls at each point.

**Screenshot** — a single image; can be taken manually at any point, or automatically on failure.

**Video** — a full screen recording of the test execution; useful for a quick visual overview, though less detailed than a trace for actually diagnosing *why* something failed.

**Common capture strategies:**
- `on-first-retry` — only capture (trace/video) when a test fails and gets retried — avoids overhead on tests that pass normally, while still capturing evidence for genuinely flaky/failing ones.
- `retain-on-failure` — keep video only for tests that ultimately failed, discard for passing ones.
- `on` — always capture (higher overhead, useful for short debugging sessions, not typical for full CI runs).

## Example

**Python — capturing on failure via a fixture:**
```python
# conftest.py
import pytest

@pytest.fixture(autouse=True)
def trace_on_failure(page, request):
    page.context.tracing.start(screenshots=True, snapshots=True, sources=True)
    yield
    if request.node.rep_call.failed:
        page.context.tracing.stop(path=f"traces/{request.node.name}.zip")
        page.screenshot(path=f"screenshots/{request.node.name}.png")
    else:
        page.context.tracing.stop()
```

```bash
# View a captured trace interactively
playwright show-trace traces/test_checkout_fails.zip
```

**TypeScript — configured declaratively (no manual fixture needed):**
```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
});
```

```bash
# View a captured trace interactively
npx playwright show-trace test-results/checkout-fails/trace.zip
```

Because TypeScript's Playwright Test runner has this built in, most teams don't need custom fixture code at all — it's config-only, one of the ergonomic advantages of the native test runner over assembling the equivalent manually in pytest.

## Production Considerations

- Use `on-first-retry` / `retain-on-failure` strategies rather than capturing everything for every test — full tracing/video on every passing test adds meaningful storage and I/O overhead at scale with little debugging value, since passing tests rarely need forensic investigation.
- Upload trace/screenshot/video artifacts to CI's artifact storage (GitHub Actions artifacts, etc.) as part of the pipeline — a trace captured but never retrievable after the CI run ends is useless for debugging a failure days later.
- Traces can get large (DOM snapshots for every action) — set a reasonable retention policy for CI artifacts rather than keeping every trace indefinitely.

## Common Pitfalls

- Only capturing a screenshot on failure and skipping trace capture — a screenshot shows the final state but not the sequence of actions/network calls that led there, which is often what's actually needed to diagnose the root cause.
- Capturing traces for every test unconditionally, adding unnecessary overhead and storage cost without a matching debugging benefit for tests that pass reliably.
- Not wiring CI to upload/preserve these artifacts, so they exist only transiently on the CI runner and are unrecoverable once the job finishes — this is a pipeline configuration step easy to forget when first setting up CI.
- Relying on video alone for debugging — it shows what happened visually but lacks the DOM/network inspection detail the trace viewer provides, making root-cause diagnosis much slower.

## Interview Notes

- Be ready to explain what the trace viewer shows that a screenshot doesn't — DOM snapshots per action, network activity, console logs — and why that matters for CI debugging specifically.
- Understand the difference between `on-first-retry`, `retain-on-failure`, and `on` capture strategies, and be able to justify a sensible default for a real CI pipeline.
- Be able to describe the full loop: a test fails in CI → trace/screenshot/video captured → uploaded as a CI artifact → downloaded and inspected via the trace viewer — this end-to-end picture is what interviewers are really checking for.

## References

- [Playwright — Trace Viewer (Python)](https://playwright.dev/python/docs/trace-viewer-intro)
- [Playwright — Trace Viewer (Node.js)](https://playwright.dev/docs/trace-viewer-intro)
- [Playwright — Screenshots (Node.js)](https://playwright.dev/docs/screenshots)