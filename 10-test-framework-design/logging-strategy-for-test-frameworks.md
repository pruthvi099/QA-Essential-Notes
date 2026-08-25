# Logging Strategy for Test Frameworks

## What It Is

This note covers deliberate logging design in a test framework — structured log levels (debug, info, warn, error), deciding what's actually worth logging, and how logging complements (rather than duplicates) the artifact-based debugging tools covered in [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md). Good logging answers "what was the framework doing, in what order, right before this failed" — a different, complementary question from what a trace/screenshot answers.

## Why It Matters

- A trace shows *what the browser did*; logs show *what the test framework/code was doing* — including logic that happens before/between/around browser actions (data setup, conditional branches, retry attempts) that a trace alone doesn't fully capture.
- Poor logging (too sparse, too noisy, or unstructured) makes CI failure investigation slower — either there's not enough context to understand what happened, or there's so much noise that finding the relevant line is its own investigation.
- This is a practical, everyday framework design decision — every framework needs *some* logging strategy, and doing it deliberately (rather than ad-hoc `console.log()` scattered inconsistently) is a meaningful maintainability difference at scale.

## How It Works

**Standard log levels, and what belongs at each:**
- **Debug** — fine-grained detail useful only when actively investigating something specific (exact locator values, raw API response bodies) — typically off by default, enabled only when needed.
- **Info** — the normal, expected flow of execution (test started, navigating to X, action Y performed) — this is what the driver wrapper's logging from [Abstraction Layers & Driver Wrappers](./abstraction-layers-and-driver-wrappers.md) typically produces.
- **Warn** — something unexpected but non-fatal happened (a retry occurred, a fallback path was taken) — worth noticing but didn't fail the test.
- **Error** — something that caused (or contributed to) a test failure — should include enough context to start investigating without needing to reproduce the failure first.

**What's worth logging in a test framework specifically:**
- Test setup/teardown boundaries (which test is running, what data it created).
- Key actions and their targets (matching the driver wrapper pattern).
- Retry attempts and their outcomes (connects to [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md) — logs are often the first clue distinguishing genuine flakiness from a real bug).
- API calls made for test data setup, including status codes.

## Example

**A structured logger used consistently through the driver wrapper and test fixtures:**
```typescript
// core/Logger.ts
enum LogLevel { DEBUG, INFO, WARN, ERROR }

class Logger {
  constructor(private context: string, private level: LogLevel = LogLevel.INFO) {}

  debug(message: string, data?: unknown) {
    if (this.level <= LogLevel.DEBUG) {
      console.log(`[DEBUG] [${this.context}] ${message}`, data ?? '');
    }
  }

  info(message: string) {
    if (this.level <= LogLevel.INFO) {
      console.log(`[INFO] [${this.context}] ${message}`);
    }
  }

  warn(message: string) {
    console.warn(`[WARN] [${this.context}] ${message}`);
  }

  error(message: string, error?: unknown) {
    console.error(`[ERROR] [${this.context}] ${message}`, error ?? '');
  }
}

export { Logger, LogLevel };
```

**Applying it across setup, actions, and retries — producing a genuinely useful failure-investigation trail:**
```typescript
// fixtures/authenticated-page.ts
const logger = new Logger('AuthFixture');

export const test = base.extend<{ authenticatedPage: Page }>({
  authenticatedPage: async ({ page }, use) => {
    logger.info('Setting up authenticated session via API login');
    const response = await page.request.post('/api/login', {
      data: { email: 'test@example.com', password: 'Pass@123' },
    });
    logger.info(`Login API response status: ${response.status()}`);

    if (!response.ok()) {
      logger.error('Login API call failed during test setup', await response.text());
      throw new Error('Failed to authenticate test session');
    }

    await use(page);
  },
});
```

**A realistic failure investigation, showing what the resulting log trail provides beyond a bare stack trace:**
```text
[INFO] [AuthFixture] Setting up authenticated session via API login
[INFO] [AuthFixture] Login API response status: 200
[INFO] [TestDriver] Navigating to: /checkout
[INFO] [TestDriver] Filling "Discount Code field" with: SAVE10
[WARN] [TestDriver] Retry attempt 1 for click on "Apply Discount button" — element not yet stable
[INFO] [TestDriver] Clicking: Apply Discount button
[ERROR] [TestDriver] Failed to click: Confirm Order button
  TimeoutError: locator.click: Timeout 5000ms exceeded

# This trail immediately shows: auth succeeded, discount code WAS
# applied (after one retry — worth noting even though it "succeeded"),
# and the actual failure was on a LATER, seemingly unrelated action —
# valuable sequencing context a stack trace alone wouldn't convey
```

## Production Considerations

- Set the default log level to `INFO` for CI runs (enough context to investigate most failures without excessive noise) and allow `DEBUG` to be enabled selectively (an environment variable) when deeper investigation of a specific, hard-to-diagnose issue is needed.
- Correlate logs with the specific test that produced them (as the `[AuthFixture]`/`[TestDriver]` context tags above do) — in a parallel-execution context (see [Parallel Execution](../02-automation-python-playwright/parallel-execution.md)), interleaved logs from multiple simultaneous tests without clear attribution become very difficult to untangle.
- Logs and artifacts (traces, screenshots) are complementary, not redundant — a mature framework surfaces both together in its CI report (see [Test Artifacts & Reports in CI](../07-ci-cd/test-artifacts-and-reports-in-ci.md)), since each answers a different part of "what happened."

## Common Pitfalls

- Logging too sparsely, leaving a failure investigation with no context beyond a bare stack trace — the exact gap deliberate logging strategy is meant to close.
- Logging too much at the default level (excessive debug-level noise always on), making genuine signal hard to find amid the volume — this often leads teams to ignore logs entirely, defeating their purpose.
- Using inconsistent, ad-hoc `console.log()` calls scattered throughout the codebase instead of a structured logger with consistent levels/formatting — this makes logs hard to filter, search, or correlate across a parallel test run.
- Not including test/context identification in log lines, making it impossible to tell which of several parallel tests produced a given log line during investigation.

## Interview Notes

- Be ready to explain what logs provide that traces/screenshots don't (and vice versa) — the complementary relationship is the core, expected insight, not "logs are just another debugging artifact."
- Understand the standard log level hierarchy (debug/info/warn/error) and be able to give framework-specific examples of what belongs at each level.
- Be able to describe why log correlation (attributing log lines to a specific test) matters specifically under parallel execution — a practical detail that shows real experience investigating CI failures at scale.

## References

- [Playwright — Debug Tests (Node.js)](https://playwright.dev/docs/debug)
- [The Twelve-Factor App — Logs](https://12factor.net/logs)