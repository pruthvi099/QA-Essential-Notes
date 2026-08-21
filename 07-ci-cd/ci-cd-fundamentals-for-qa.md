# CI/CD Fundamentals for QA

## What It Is

**Continuous Integration (CI)** is the practice of automatically building and testing code every time it's pushed, catching integration issues early. **Continuous Delivery** extends this by automatically preparing every passing build for release (but requiring a manual approval to actually deploy). **Continuous Deployment** goes further still, automatically deploying every passing build straight to production with no manual gate. This note establishes where automated testing fits into each stage — the foundation for every other note in this folder.

## Why It Matters

- Testing strategy fundamentally depends on which of these three models a team uses — the same test suite plays a very different role in a pipeline with a manual release gate versus one that deploys straight to production on every green build.
- This directly connects back to [SDLC Models](../00-start-here/sdlc-models.md) — CI/CD is the concrete, tooling-level expression of the "continuous testing" concept introduced there for Agile/DevOps teams.
- Nearly every modern SDET role assumes working fluency with CI/CD concepts — pipelines are where automated tests actually deliver their value (fast feedback, release confidence), not just where they happen to run.

## How It Works

**The core distinction:**

| | CI | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Build & test automatically | Yes | Yes | Yes |
| Release-ready artifact prepared automatically | No (not the focus) | Yes | Yes |
| Deploy to production | Manual, separate step | Manual approval gate | Fully automatic |

**Where testing fits at each stage of a typical pipeline:**
1. **Commit stage** — fast unit/lint checks run on every push, giving near-immediate feedback (see [Types of Testing](../00-start-here/types-of-testing.md) for smoke/sanity/regression tiering, applied to CI staging in [Smoke vs. Regression in CI](./smoke-vs-regression-in-ci.md)).
2. **Build & integration stage** — the application is built and integration/API tests run against it.
3. **Acceptance/E2E stage** — broader end-to-end tests run, often against a deployed staging environment.
4. **Release/deploy stage** — for Continuous Delivery/Deployment, this is gated by all prior stages passing; for Continuous Deployment specifically, a passing pipeline triggers an automatic production release.

The further along this pipeline a test runs, the slower and more expensive it typically is — this is the same testing-pyramid logic from [Levels of Testing](../00-start-here/levels-of-testing.md), now expressed as pipeline staging rather than test architecture.

## Example

A conceptual pipeline structure showing where different test types run, tying together concepts from across the repo:

```yaml
# Conceptual pipeline stages (implementation-specific YAML covered in
# GitHub Actions / Jenkins notes)

stages:
  - name: commit
    trigger: every push
    runs:
      - lint
      - unit tests
    duration_target: "< 2 minutes"

  - name: build_and_integration
    trigger: every push
    runs:
      - build application
      - API/integration tests (see 04-api-testing/)
    duration_target: "< 10 minutes"

  - name: acceptance
    trigger: every pull request
    runs:
      - smoke-tagged E2E tests (see 02-automation-python-playwright/)
    duration_target: "< 15 minutes"

  - name: full_regression
    trigger: nightly schedule, or pre-release
    runs:
      - full E2E regression suite
      - cross-browser matrix (see 03-typescript-playwright/)
    duration_target: "acceptable up to 1+ hour, runs less frequently"

  - name: deploy
    trigger: manual approval (Continuous Delivery) OR automatic (Continuous Deployment)
    depends_on: all prior stages passing
```

## Production Considerations

- The choice between Continuous Delivery (manual approval) and Continuous Deployment (fully automatic) is a real risk/velocity trade-off, not just a technical configuration — it should be a deliberate decision involving the whole team, not something a testing framework choice quietly implies on its own.
- Fast feedback stages (commit, build/integration) should be optimized aggressively for speed, since they run on every single push — slow feedback here is what erodes developer trust in CI and encourages skipping/ignoring failures.
- As a team moves toward Continuous Deployment, automated test coverage and quality gates (see [Quality Gates & Build Failures](./quality-gates-and-build-failures.md)) become the *only* safety net before production — this raises the stakes on test suite reliability considerably compared to a model with a human manual-approval checkpoint.

## Common Pitfalls

- Confusing Continuous Delivery and Continuous Deployment — they're often used loosely/interchangeably in casual conversation, but the distinction (manual approval vs. fully automatic) matters for understanding a team's actual risk posture and testing requirements.
- Running the full, slow regression suite at the commit stage "to be thorough," destroying fast feedback for every single push — this is a common, well-intentioned mistake with real productivity cost.
- Treating CI/CD as purely a DevOps/infrastructure concern with no input needed from QA/SDET — pipeline test staging decisions directly shape testing strategy and should be a collaborative design, not something QA is handed after the fact.
- Not having any pipeline stage between "runs on every push" and "runs nightly" — a middle acceptance-test tier (as shown for pull requests above) is often what actually catches meaningful regressions before merge, without the cost of full regression on every single commit.

## Interview Notes

- Be ready to precisely distinguish CI, Continuous Delivery, and Continuous Deployment — a foundational, frequently asked distinction that's easy to answer vaguely but should be stated precisely.
- Understand how test staging (what runs at each pipeline stage) maps to the testing pyramid/levels concept — this shows CI/CD understanding as an extension of core testing principles, not a separate, disconnected topic.
- Be able to describe how testing requirements change as a team moves from CI alone toward full Continuous Deployment — specifically, why test suite reliability becomes more critical as manual gates are removed.

## References

- [Martin Fowler — Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
- [Atlassian — CI/CD Pipeline](https://www.atlassian.com/continuous-delivery/continuous-integration)