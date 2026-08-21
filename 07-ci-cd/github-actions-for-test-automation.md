# GitHub Actions for Test Automation

## What It Is

GitHub Actions is GitHub's built-in CI/CD platform, using YAML workflow files to define automated jobs triggered by repository events (pushes, pull requests, schedules). It's one of the most common places SDETs actually configure and run test suites, given GitHub's widespread use for source control — making it a practical, high-value tool to be hands-on comfortable with.

## Why It Matters

- Workflow configuration is often the SDET's direct responsibility, not just a DevOps concern — deciding what runs when, how artifacts are captured, and how failures are reported is testing strategy expressed as pipeline configuration (see [CI/CD Fundamentals for QA](./ci-cd-fundamentals-for-qa.md)).
- GitHub Actions' tight integration with pull requests (inline status checks, required checks before merge) makes it a common way quality gates (see [Quality Gates & Build Failures](./quality-gates-and-build-failures.md)) are actually enforced in practice, not just discussed abstractly.
- Practical GitHub Actions fluency is a frequently expected, concrete skill in SDET job postings and interviews — being able to sketch a working workflow file live is a common practical exercise.

## How It Works

**Core concepts:**
- **Workflow** — a YAML file in `.github/workflows/` defining automated jobs.
- **Trigger (`on`)** — what causes the workflow to run: `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual trigger).
- **Job** — a set of steps that run on a specified runner (a VM); multiple jobs run in parallel by default unless dependencies are declared.
- **Step** — an individual command or reusable action within a job.
- **Matrix strategy** — runs the same job multiple times with different parameter combinations (e.g., multiple browsers, multiple Node/Python versions) — the CI-level equivalent of the parameterization covered in [Config-Driven Test Parameterization](../03-typescript-playwright/typescript-config-driven-test-parameterization.md).

## Example

A realistic GitHub Actions workflow running a Playwright TypeScript suite on pull requests, with a browser matrix and artifact upload:

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
      fail-fast: false   # let other browsers keep running even if one fails

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps ${{ matrix.browser }}

      - name: Run E2E tests
        run: npx playwright test --project=${{ matrix.browser }}
        env:
          BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          TEST_ENV: staging

      - name: Upload test report
        if: always()   # upload even if tests failed — the report is most needed then
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.browser }}
          path: playwright-report/
          retention-days: 14
```

A separate scheduled workflow for the full nightly regression suite, distinct from the faster PR-triggered smoke run:

```yaml
# .github/workflows/nightly-regression.yml
name: Nightly Full Regression

on:
  schedule:
    - cron: '0 2 * * *'   # 2 AM UTC daily
  workflow_dispatch: {}    # also allow manual triggering

jobs:
  full_regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx playwright install --with-deps
      - name: Run full regression suite
        run: npx playwright test --grep @regression
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: nightly-regression-report
          path: playwright-report/
```

## Production Considerations

- Use `if: always()` on artifact upload steps specifically — by default, subsequent steps are skipped once a step fails, meaning the test report (most valuable exactly when tests fail) wouldn't be uploaded without this.
- Store credentials and environment-specific values in GitHub's encrypted Secrets (`secrets.STAGING_BASE_URL` above), never hardcoded in the workflow YAML — this follows the same principle as [Secrets & Environment Management in CI](./secrets-and-environment-management-in-ci.md).
- Cache dependencies (`cache: 'npm'` above, or an equivalent for pip) to meaningfully speed up repeated CI runs — dependency installation is often a significant, avoidable portion of total pipeline time.

## Common Pitfalls

- Forgetting `if: always()` on report/artifact upload steps, losing exactly the debugging information (traces, screenshots) needed when a test actually fails — see [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md).
- Running the full regression suite on every single pull request instead of a fast smoke subset, slowing PR feedback significantly (see [Smoke vs. Regression in CI](./smoke-vs-regression-in-ci.md) for the staging strategy this avoids).
- Not setting `fail-fast: false` on a matrix strategy when independent failure information matters (e.g., wanting to see failures on all three browsers, not just the first one that fails) — the default `fail-fast: true` cancels remaining matrix jobs on the first failure.
- Hardcoding secrets or environment URLs directly in the workflow YAML file, which is committed to version control and potentially visible to anyone with repository read access.

## Interview Notes

- Be ready to sketch a basic GitHub Actions workflow file live, including trigger, checkout, dependency install, and test execution steps — a common, practical exercise.
- Understand what a matrix strategy is and why it's useful for cross-browser/cross-version testing — connects directly to parameterization concepts covered elsewhere in the repo.
- Be able to explain why `if: always()` matters specifically for artifact upload steps — a small, specific detail that reveals real hands-on CI configuration experience.

## References

- [GitHub Actions — Documentation](https://docs.github.com/en/actions)
- [GitHub Actions — Using a Matrix for Your Jobs](https://docs.github.com/en/actions/using-jobs/using-a-build-matrix-for-your-jobs)
- [Playwright — CI with GitHub Actions](https://playwright.dev/docs/ci-intro)