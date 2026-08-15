# Custom Reporters & HTML Report

## What It Is

Playwright Test ships with built-in reporters (`list`, `dot`, `html`, `json`, `junit`) that control how test results are output, and supports writing fully custom reporters via the `Reporter` interface for integrating with external systems (Slack notifications, custom dashboards, test management tools). The built-in **HTML reporter** specifically produces an interactive report with pass/fail status, traces, screenshots, and videos linked per test — the default way most teams review a CI run's results.

This is a TypeScript/Playwright-Test-specific capability; pytest has its own separate reporting ecosystem (`pytest-html`, JUnit XML via `--junitxml`) that works differently under the hood, which is why this note focuses on the native Playwright Test reporter API.

## Why It Matters

- A CI run's results need to be reviewable by humans (developers checking a PR) and machines (CI systems parsing pass/fail for merge gates) — reporters serve both needs, and choosing the right one(s) for each context matters.
- The HTML reporter's integration with traces/screenshots/videos (see [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md)) makes it the single most useful artifact for debugging a CI failure without re-running anything locally.
- Custom reporters are a practical, real capability SDETs are asked to build — e.g., posting a Slack message when the nightly suite fails, or feeding results into an internal test-metrics dashboard (connecting to [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md)).

## How It Works

**Multiple reporters can run simultaneously** — commonly, a human-readable one (`html`) plus a machine-readable one (`junit` for CI integration, `json` for custom tooling):

```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['html', { open: 'never' }],
    ['junit', { outputFile: 'results/junit.xml' }],
    ['json', { outputFile: 'results/results.json' }],
  ],
});
```

**A custom reporter** implements the `Reporter` interface, hooking into lifecycle events (`onTestEnd`, `onEnd`, etc.):

```typescript
import type { Reporter, TestCase, TestResult, FullResult } from '@playwright/test/reporter';

class SlackReporter implements Reporter {
  private failures: string[] = [];

  onTestEnd(test: TestCase, result: TestResult) {
    if (result.status === 'failed') {
      this.failures.push(test.title);
    }
  }

  async onEnd(result: FullResult) {
    if (this.failures.length > 0) {
      await notifySlack(`${this.failures.length} test(s) failed: ${this.failures.join(', ')}`);
    }
  }
}

export default SlackReporter;
```

## Example

A realistic multi-reporter CI setup, plus a custom reporter that summarizes results beyond what built-in reporters provide:

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  reporter: process.env.CI
    ? [
        ['html', { open: 'never' }],
        ['junit', { outputFile: 'results/junit.xml' }],
        ['./reporters/slack-reporter.ts'],
      ]
    : [['list']],   // fast, minimal output for local runs
});
```

```typescript
// reporters/slack-reporter.ts
import type { Reporter, TestCase, TestResult, FullResult } from '@playwright/test/reporter';

interface FailureSummary {
  title: string;
  file: string;
  error: string;
}

export default class SlackReporter implements Reporter {
  private failures: FailureSummary[] = [];

  onTestEnd(test: TestCase, result: TestResult) {
    if (result.status === 'failed' || result.status === 'timedOut') {
      this.failures.push({
        title: test.title,
        file: test.location.file,
        error: result.error?.message ?? 'Unknown error',
      });
    }
  }

  async onEnd(result: FullResult) {
    if (this.failures.length === 0) return;

    const message = [
      `❌ ${this.failures.length} test(s) failed (run status: ${result.status})`,
      ...this.failures.map(f => `• ${f.title} (${f.file})`),
    ].join('\n');

    await fetch(process.env.SLACK_WEBHOOK_URL!, {
      method: 'POST',
      body: JSON.stringify({ text: message }),
    });
  }
}
```

## Production Considerations

- Use `open: 'never'` for the HTML reporter in CI — it defaults to opening a local browser automatically, which is meaningless (and can even hang) on a headless CI runner; upload the generated HTML report as a CI artifact instead, for humans to open afterward.
- JUnit XML output is the standard format most CI systems (GitHub Actions, Jenkins, GitLab CI) natively parse for showing inline pass/fail status on a PR — include it even if the HTML reporter is your primary human-facing report.
- Custom reporters should fail gracefully (e.g., a Slack webhook call failing shouldn't crash the whole CI run) — wrap external calls in their own try/catch so reporting infrastructure issues don't masquerade as test failures.

## Common Pitfalls

- Using only the `html` reporter in CI without a machine-readable format (`junit`/`json`) — this leaves CI systems unable to natively show pass/fail status inline on a PR, since they need a format they can parse.
- Leaving the HTML reporter's default `open` behavior enabled in CI, which can attempt to launch a browser on a headless runner and hang the pipeline.
- Writing a custom reporter that throws an unhandled error inside `onEnd`/`onTestEnd` — this can crash the entire Playwright Test process, obscuring the actual test results underneath a reporter bug.
- Not uploading the HTML report as a CI artifact — the report is generated but immediately lost once the CI job finishes, exactly the failure mode discussed for traces in [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md).

## Interview Notes

- Be ready to explain why a CI pipeline typically needs more than one reporter simultaneously (human-readable + machine-readable) rather than just one.
- Understand the basic shape of the `Reporter` interface (`onTestEnd`, `onEnd`) and be able to describe a practical custom reporter use case (Slack/Teams notification, internal dashboard feed).
- Be able to explain what makes the HTML reporter specifically valuable for CI debugging (linked traces/screenshots/videos per test) compared to a plain pass/fail log.

## References

- [Playwright — Reporters (Node.js)](https://playwright.dev/docs/test-reporters)
- [Playwright — Custom Reporters (Node.js)](https://playwright.dev/docs/test-reporters#custom-reporters)