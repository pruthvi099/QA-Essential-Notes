# 07 — CI/CD

Integrating test automation into delivery pipelines — from fundamentals to platform-specific configuration, staging strategy, artifacts, secrets, quality gates, and scaling. Read [02-automation-python-playwright](../02-automation-python-playwright/) and [03-typescript-playwright](../03-typescript-playwright/) first for the test suites these pipelines actually run.

## Notes

1. [CI/CD Fundamentals for QA](./ci-cd-fundamentals-for-qa.md) — CI vs. Continuous Delivery vs. Continuous Deployment
2. [GitHub Actions for Test Automation](./github-actions-for-test-automation.md) — Building a workflow to run a Playwright suite
3. [Jenkins for Test Automation](./jenkins-for-test-automation.md) — Declarative pipelines for enterprise CI
4. [Pull Request Test Triggers](./pull-request-test-triggers.md) — What runs on PR vs. merge vs. schedule
5. [Smoke vs. Regression in CI](./smoke-vs-regression-in-ci.md) — Tagging and pipeline-stage mechanics for test tiering
6. [Test Artifacts & Reports in CI](./test-artifacts-and-reports-in-ci.md) — Publishing traces, reports, and JUnit results
7. [Secrets & Environment Management in CI](./secrets-and-environment-management-in-ci.md) — Credential handling and multi-environment config
8. [Quality Gates & Build Failures](./quality-gates-and-build-failures.md) — What blocks a merge/deploy vs. what warns
9. [Parallel & Sharded CI Execution](./parallel-and-sharded-ci-execution.md) — Scaling a suite across CI runners
10. [Flaky Test Quarantine in CI](./flaky-test-quarantine-in-ci.md) — Pipeline-level workflows for managing flakiness at scale

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — the test suites these pipelines execute
- [00 — Start Here](../00-start-here/) — entry/exit criteria and SDLC models this folder's quality gates enforce