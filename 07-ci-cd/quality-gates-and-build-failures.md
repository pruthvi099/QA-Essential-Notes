# Quality Gates & Build Failures

## What It Is

A quality gate is a defined, enforced condition that must be satisfied before code can merge or deploy — a failing gate blocks the pipeline, while a warning surfaces information without blocking. This note covers deciding *what actually blocks* (required checks, coverage thresholds, zero critical defects) versus what merely warns, tying together [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md)'s conceptual criteria with concrete pipeline enforcement mechanisms.

## Why It Matters

- A quality gate is only meaningful if it's actually enforced — a "required" check that can be bypassed with an admin override, or a coverage threshold nobody actually reads, provides false confidence rather than real protection.
- Setting gates too strictly (blocking merges on flaky, non-deterministic checks) trains teams to routinely override or disable them, which defeats their purpose entirely; setting them too loosely lets real regressions through — this calibration is a genuine, ongoing design decision.
- This is where all the earlier CI/CD notes converge into an enforcement mechanism — trigger design, tiering, and artifacts all exist in service of quality gates actually being able to reliably decide "is this safe to merge/deploy."

## How It Works

**Common quality gate criteria:**
1. **Required status checks** — specific CI jobs (smoke tests, lint, build) that must pass before a PR can merge, enforced at the platform level (GitHub branch protection rules, Jenkins pipeline gating).
2. **Code coverage thresholds** — a minimum percentage of code covered by tests, though this needs the same caution against over-reliance discussed in [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md) — coverage alone isn't a full quality signal.
3. **Zero critical/blocker defects** — no known critical, unresolved bugs in the affected area (often enforced via a manual check or an issue-tracker integration, less commonly a fully automated gate).
4. **Security scan results** — no newly introduced high-severity vulnerabilities from dependency/SAST scanning.

**What should block vs. warn:**
- **Block**: smoke test failures, build failures, security vulnerabilities above a defined severity, required lint/type errors.
- **Warn (don't block)**: coverage dropping slightly (unless it crosses a hard floor), non-critical/known-flaky test failures already being tracked, style/formatting suggestions.

## Example

**GitHub branch protection configuration (conceptual, set via repository settings, not YAML) enforcing required checks:**
```text
Branch protection rule for `main`:
  Require status checks to pass before merging:
    ✓ smoke_tests (from PR Checks workflow)
    ✓ lint
    ✓ build
  Require branches to be up to date before merging: enabled
  Do NOT require: full_regression (runs post-merge, not PR-blocking —
    see Pull Request Test Triggers)
  Restrict who can bypass these requirements: repository admins only,
    with the expectation that bypassing is logged and reviewed
```

**A pipeline step that explicitly fails the build on a defined threshold, versus one that only warns:**
```yaml
- name: Check code coverage threshold
  run: |
    COVERAGE=$(npx nyc report --reporter=text-summary | grep "Lines" | grep -oP '\d+(?=%)')
    echo "Current coverage: ${COVERAGE}%"

    if [ "$COVERAGE" -lt 60 ]; then
      echo "::error::Coverage ${COVERAGE}% is below the 60% HARD FLOOR — blocking merge"
      exit 1
    elif [ "$COVERAGE" -lt 75 ]; then
      echo "::warning::Coverage ${COVERAGE}% is below the 75% target, but above the hard floor — not blocking"
    fi
```

A deliberately explicit distinction between a genuine gate and a tracked, non-blocking known issue:
```yaml
- name: Run smoke tests
  run: npx playwright test --grep @smoke
  # No continue-on-error — a smoke test failure BLOCKS the pipeline,
  # by design, since these represent critical-path breakage

- name: Run known-flaky quarantined tests
  run: npx playwright test --grep @quarantined
  continue-on-error: true   # explicitly does NOT block — see Flaky
                              # Test Quarantine in CI for the full workflow
                              # this connects to
```

## Production Considerations

- Every blocking gate needs a documented, low-friction override path for genuine emergencies (a critical production hotfix that can't wait for a flaky unrelated check) — without one, teams find informal, less auditable ways around the gate instead, which is worse for both safety and visibility.
- Gate criteria should be revisited periodically against real data — if a "required" check has a track record of failing for reasons unrelated to real regressions (environmental flakiness), it's actively harming trust in the gate system and needs fixing or reclassifying, not just tolerated.
- Communicate quality gate criteria clearly to the whole engineering team, not just QA/SDET — a gate that developers don't understand the reasoning for gets resented and worked around; one they understand becomes a shared, respected standard.

## Common Pitfalls

- Blocking merges on inherently flaky or environmentally unstable checks, training the team to routinely bypass or disable the gate — this is one of the most damaging patterns in CI/CD, since it erodes trust in *all* gates, not just the flaky one.
- Setting a coverage threshold as a hard blocking gate without the nuance from [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md) — coverage percentage alone, without considering what's actually covered, can be gamed or misleading as a sole blocking criterion.
- Having gates that exist in documentation but aren't actually technically enforced (e.g., a "required" check that isn't configured as a GitHub branch protection requirement) — this is a false sense of safety, not real protection.
- Not distinguishing "must block" from "should warn" clearly, leading to either an overly permissive pipeline (nothing really blocks) or an overly restrictive one (everything blocks, including things that shouldn't).

## Interview Notes

- Be ready to design a quality gate strategy for a hypothetical pipeline — what blocks, what warns, and why — a common, practical CI/CD interview question that ties together most of this folder's concepts.
- Understand why blocking on flaky checks is specifically damaging (beyond just being annoying) — it trains bypass behavior that undermines every other gate, not just the flaky one.
- Be able to explain the relationship between quality gates here and the entry/exit criteria concept from [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md) — gates are the CI/CD-level, automated enforcement of those same criteria.

## References

- [GitHub — About Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [SonarQube — Quality Gates](https://docs.sonarsource.com/sonarqube/latest/user-guide/quality-gates/)