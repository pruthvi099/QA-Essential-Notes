# Test Artifacts & Reports in CI

## What It Is

This note covers systematically publishing test outputs — traces, screenshots, videos, HTML reports, JUnit XML — as retrievable CI artifacts, connecting the artifact-generation concepts from [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md) and [Custom Reporters & HTML Report](../03-typescript-playwright/custom-reporters-and-html-report.md) with the pipeline-level mechanics (GitHub Actions' `upload-artifact`, Jenkins' `archiveArtifacts`/`junit`) that actually make them accessible after a CI run finishes.

## Why It Matters

- An artifact generated locally during a CI run but never explicitly uploaded is effectively lost the moment the runner is torn down — this is one of the most common, avoidable gaps in CI pipelines, and directly undermines all the debugging value those artifacts were built to provide.
- Different audiences need different artifact formats simultaneously: developers debugging a failure want the HTML report with traces; the CI system itself needs machine-parseable JUnit XML to show inline PR status — a pipeline needs both, not one or the other.
- This closes the loop between test execution and actual usability of results — a suite that runs perfectly but produces no accessible, well-organized output when it fails delivers far less real value than one with a smaller test count but excellent failure visibility.

## How It Works

**Categories of CI test artifacts:**
1. **Human-readable reports** — HTML reports (Playwright's built-in, `pytest-html`) with embedded traces/screenshots/videos, meant for a person investigating a failure.
2. **Machine-readable results** — JUnit XML, JSON — meant for the CI system itself to parse and display inline (PR status checks, dashboards).
3. **Raw debugging artifacts** — trace files, screenshots, videos, logs — the deepest level of detail, usually linked from within the HTML report.

**Retention strategy** — artifacts add storage cost over time; most CI platforms support configurable retention periods (e.g., 14–30 days), balancing "long enough to investigate a delayed bug report" against unbounded storage growth.

## Example

**GitHub Actions — a complete artifact strategy combining all three categories (extending the earlier [GitHub Actions](./github-actions-for-test-automation.md) example):**

```yaml
- name: Run E2E tests
  run: npx playwright test --reporter=html,junit
  env:
    PLAYWRIGHT_JUNIT_OUTPUT_NAME: results/junit.xml

- name: Upload HTML report (human-readable, includes traces/screenshots/videos)
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-html-report
    path: playwright-report/
    retention-days: 14

- name: Upload JUnit results (machine-readable, for PR status)
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: junit-results
    path: results/junit.xml
    retention-days: 30

- name: Publish test results to PR
  if: always()
  uses: dorny/test-reporter@v1
  with:
    name: E2E Test Results
    path: results/junit.xml
    reporter: java-junit   # JUnit format is broadly reusable across ecosystems
```

**Jenkins — the equivalent artifact strategy (extending the earlier [Jenkins](./jenkins-for-test-automation.md) example):**
```groovy
post {
    always {
        // Machine-readable: powers Jenkins' built-in test trend UI
        junit 'results/junit.xml'

        // Human-readable: the full HTML report with linked traces
        archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true

        // Explicit retention configuration at the JOB level (not per-build),
        // typically set via Jenkins' "Discard old builds" job configuration
        // rather than inline in the Jenkinsfile
    }
}
```

A short investigation workflow this setup enables — the actual payoff of doing this correctly:
```text
1. PR shows a failed check (from the JUnit-based inline status)
2. Developer clicks through to the CI run
3. Downloads/opens the HTML report artifact
4. Report links directly to the trace file for the failed test
5. Opens the trace viewer, sees the exact DOM state and network
   activity at the moment of failure — no local reproduction needed
   to start investigating
```

## Production Considerations

- Set retention periods deliberately per artifact type — JUnit XML (small, useful for longer-term trend analysis) can reasonably be kept longer than full HTML reports with videos (large, mainly useful for near-term debugging of a specific recent failure).
- `if: always()` (GitHub Actions) / `post { always {...} } ` (Jenkins) must wrap every artifact-publishing step — this is worth double-checking specifically, since it's the single most common reason artifacts silently go missing exactly when they're needed most (on failure).
- For teams at real scale, consider whether raw artifact storage (screenshots/videos/traces) needs its own retention/cost policy separate from the CI platform's default — video artifacts in particular can accumulate significant storage cost if left unmanaged.

## Common Pitfalls

- Omitting `if: always()` / `post { always {...} }`, so artifacts only get published on success — exactly backwards from when they're actually needed.
- Publishing only a human-readable report or only machine-readable results, but not both — this either leaves the CI system unable to show inline PR status, or leaves developers without a usable debugging report.
- Not setting any retention policy, letting artifacts accumulate indefinitely and driving up storage costs without corresponding investigative value for very old runs.
- Generating rich artifacts (traces, videos) but never actually linking or surfacing them from the CI UI/PR in a way developers will actually notice and use — artifacts that exist but aren't discoverable provide much less real value.

## Interview Notes

- Be ready to explain why a CI pipeline needs both human-readable and machine-readable test output formats simultaneously, not just one.
- Understand the specific, common failure mode of forgetting `if: always()`/`post always` on artifact steps, and why it's especially damaging (artifacts missing exactly when needed).
- Be able to describe the full debugging workflow a well-configured artifact strategy enables — from a failed PR check to opening a trace viewer — showing you understand the end-to-end value, not just the individual configuration steps.

## References

- [GitHub Actions — Storing Workflow Data as Artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)
- [Jenkins — Recording Test Results](https://www.jenkins.io/doc/pipeline/steps/junit/)
- [Playwright — CI Best Practices](https://playwright.dev/docs/ci-intro)