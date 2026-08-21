# Pull Request Test Triggers

## What It Is

This note covers the strategic decision of *what* runs *when* in a CI pipeline — specifically, which tests trigger on a pull request versus on merge to main versus on a schedule. This is a design decision, not just a configuration detail: it directly balances fast developer feedback against thorough coverage, and gets its own note because getting it wrong (in either direction) has real, ongoing productivity cost.

## Why It Matters

- PR-triggered tests are what developers wait on before merging — slow or excessive PR checks directly and repeatedly cost developer time and encourage workarounds (skipping tests, force-merging), while too little PR coverage lets real regressions slip through to `main`.
- This is where [Smoke vs. Regression in CI](./smoke-vs-regression-in-ci.md) tiering, [CI/CD Fundamentals for QA](./ci-cd-fundamentals-for-qa.md)'s staging model, and platform-specific trigger syntax (GitHub Actions' `on: pull_request`, Jenkins' `when { changeRequest() }`) all come together into one concrete design decision an SDET is expected to own.
- Interviewers ask "what would you run on every PR vs. nightly" specifically because the answer reveals whether a candidate thinks about the cost/coverage trade-off deliberately, or just runs everything everywhere by default.

## How It Works

**Common trigger categories and what typically belongs in each:**

| Trigger | Typical content | Why |
|---|---|---|
| **Every push (any branch)** | Lint, unit tests | Fastest possible feedback, should complete in well under a few minutes |
| **Pull request opened/updated** | Smoke-tagged E2E, API tests | Fast enough to not block reviewers/merging, but covers critical paths |
| **Merge to main** | Broader integration tests, possibly a fuller regression subset | Slightly more coverage tolerance since it's not blocking an in-progress review |
| **Scheduled (nightly)** | Full regression, full cross-browser/device matrix | Not blocking anyone; thoroughness matters more than speed here |
| **Manual/on-demand** | Performance tests, exploratory automation runs | Run only when specifically needed, avoiding unnecessary resource use |

**The core design question for any given test:** "Does this test need to block a merge right now, or can it run less urgently?" — high-confidence, fast, critical-path tests belong on PRs; slower, broader, or lower-urgency tests belong on a schedule or post-merge.

## Example

**GitHub Actions — distinct triggers reflecting this tiering (extending the [GitHub Actions](./github-actions-for-test-automation.md) example with explicit reasoning):**

```yaml
# .github/workflows/pr-checks.yml — fast, blocks merge
name: PR Checks
on:
  pull_request:
    branches: [main]

jobs:
  smoke_tests:
    runs-on: ubuntu-latest
    timeout-minutes: 10   # a hard ceiling — if PR checks regularly approach
                            # this, it's a signal the tier boundary needs revisiting
    steps:
      - uses: actions/checkout@v4
      - run: npx playwright test --grep @smoke
```

```yaml
# .github/workflows/post-merge.yml — runs after merge, not blocking review
name: Post-Merge Integration
on:
  push:
    branches: [main]

jobs:
  integration_tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx playwright test --grep @regression --grep-invert @slow
```

```yaml
# .github/workflows/nightly.yml — thorough, not time-pressured
name: Nightly Full Regression
on:
  schedule:
    - cron: '0 2 * * *'

jobs:
  full_regression:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
    steps:
      - uses: actions/checkout@v4
      - run: npx playwright test --project=${{ matrix.browser }}
```

A deliberate exception worth documenting explicitly — a test that seems like it belongs on every PR, but doesn't:

```text
Decision: Full cross-browser visual regression suite (see 03-typescript-playwright/
visual-regression-testing.md) does NOT run on every PR.

Reasoning: Takes ~25 minutes across all browsers/viewports — well past
the target PR feedback window. Runs nightly instead. A LIGHTER visual
check (single browser, critical pages only) DOES run on PR, catching
the highest-risk visual regressions without the full cost.
```

## Production Considerations

- Set an explicit time target for PR-blocking checks (e.g., "under 10 minutes") and treat approaching or exceeding it as a signal to re-tier tests, not just accept the slowdown — PR check speed tends to creep upward gradually as more tests get added without deliberate re-evaluation.
- When a test is deliberately excluded from PR checks, document *why* (as in the example above) — this prevents someone later assuming the exclusion was an oversight and "fixing" it by adding a slow test back to the fast tier.
- Post-merge (not PR-blocking) failures still need clear ownership and fast response — a regression that slips past PR checks and fails post-merge should trigger prompt investigation, not be treated as lower priority just because it didn't block anyone directly.

## Common Pitfalls

- Running the entire test suite on every PR "to be safe," creating a slow feedback loop that trains developers to context-switch away and ignore CI status, or to push for merging despite a red build out of frustration.
- Under-covering PR checks so much that broken code regularly reaches `main`, pushing the cost of catching regressions later (post-merge or nightly), where it's more disruptive and harder to trace back to a specific small change.
- Not revisiting trigger/tiering decisions as the suite grows — a tiering strategy that made sense at 50 tests can become inappropriate at 500 without deliberate re-evaluation.
- Treating post-merge or nightly failures as lower priority simply because they didn't block a specific PR — a real regression is still a real regression regardless of which stage caught it.

## Interview Notes

- Be ready to design a trigger/tiering strategy for a hypothetical test suite, explaining what runs on PR vs. post-merge vs. nightly and why — a very common, practical CI/CD interview question.
- Understand the core trade-off (fast feedback vs. thorough coverage) that drives every trigger decision, and be able to articulate it clearly rather than defaulting to "run everything everywhere."
- Be able to describe how you'd notice and respond to PR check times creeping upward over time — this shows ongoing pipeline health awareness, not just initial design.

## References

- [GitHub Actions — Events that Trigger Workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [Martin Fowler — Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)