# Running Tests in Containers vs. Locally

## What It Is

This note covers the practical trade-offs between running a test suite natively on a local machine versus inside a container — when containerized execution genuinely helps (environment parity, CI consistency), when it adds friction without proportional benefit, and the specific gotchas that trip people up when a suite runs differently in a container than it did locally.

## Why It Matters

- Containerization isn't free — it adds real overhead (build time, resource allocation, debugging complexity) that's worth it in some contexts and unnecessary friction in others; treating "always containerize" or "never containerize" as a blanket rule misses genuine, situational trade-offs.
- "It passes locally but fails in the container" is a common, specific class of bug distinct from ordinary flakiness — usually rooted in genuine environment differences (screen/display handling, resource limits, networking) that are worth recognizing as a category.
- Interviewers ask about this trade-off specifically to check whether a candidate has actually run tests both ways and understands the real difference, rather than treating "containerize everything" as an unconsidered default.

## How It Works

**When containerized execution genuinely helps:**
- CI environments — guarantees the exact same environment every run, eliminating "works in CI sometimes" environment drift.
- Onboarding new team members — `docker compose up` gives a working environment instantly, without a lengthy local setup guide.
- Cross-platform consistency — a team with mixed OS laptops (macOS, Windows, Linux) gets identical test execution regardless of host OS.
- Integration testing needing multiple coordinated services (see [Docker Compose for Test Stacks](./docker-compose-for-test-stacks.md)).

**When it adds friction without proportional benefit:**
- Fast, iterative local development — the container build/rebuild cycle can slow down the tight edit-run-debug loop compared to running natively.
- Simple, small projects with few environment-sensitive dependencies — the setup overhead isn't justified if native execution is already reliable and consistent.
- Interactive debugging sessions (see [Debugging Containerized Test Failures](./debugging-containerized-test-failures.md)) — often genuinely harder inside a container than with a local GUI debugger/browser.

**Common categories of "works locally, fails in container" issues:**
1. **Headless vs. headed execution** — a container typically has no display; a test implicitly relying on a visible browser window (rare, but possible with certain configurations) fails without proper headless setup.
2. **Resource constraints** — containers often run with less CPU/memory than a developer's full local machine by default, and a suite tuned for local resources can behave differently (slower, more prone to genuine timeouts) under tighter limits.
3. **Networking differences** — `localhost` inside a container refers to the container itself, not the host machine, catching people who assume it means the same thing in both contexts (the same gotcha noted in [Docker Compose for Test Stacks](./docker-compose-for-test-stacks.md)).
4. **File path/permission differences** — a container's filesystem and user permissions can differ from local expectations, breaking tests that assume specific local paths or write access.

## Example

A concrete "works locally, fails in container" investigation, illustrating the resource-constraint category specifically:

```yaml
# The test suite ran fine locally but was intermittently timing out
# in a containerized CI run — investigation revealed the container
# had a restrictive memory limit causing slower-than-expected execution

# docker-compose.test.yml (BEFORE — no explicit limits, but the CI
# runner's default container allocation was tighter than a dev laptop)
services:
  test-runner:
    build: ./tests
    command: npx playwright test
```

```yaml
# AFTER — explicit, adequate resource allocation, plus increased
# timeouts to account for genuinely slower execution under container
# constraints, rather than assuming local-machine timing applies directly
services:
  test-runner:
    build: ./tests
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
    environment:
      # Playwright's default timeouts assumed local-machine speed;
      # explicitly widened for the container's more constrained environment
      PLAYWRIGHT_TIMEOUT: "45000"
    command: npx playwright test
```

A `localhost`-vs-service-name gotcha, a very common specific trip-up:
```python
# FAILS when run inside a container as part of a Compose stack —
# "localhost" refers to the CONTAINER itself, not the app's container
BASE_URL = "http://localhost:3000"

# CORRECT for containerized execution — reaches the actual app
# container via Compose's internal service networking
BASE_URL = "http://app:3000"

# A common practical fix: make this environment-driven so the SAME
# test code works both locally (BASE_URL=http://localhost:3000) and
# in a Compose stack (BASE_URL=http://app:3000) without code changes
import os
BASE_URL = os.environ.get("BASE_URL", "http://localhost:3000")
```

## Production Considerations

- Make environment-dependent values (base URLs, timeouts) configurable via environment variables (as shown above) rather than hardcoded — this lets the exact same test code run correctly both locally and containerized, without maintaining two separate versions.
- Explicitly set resource limits/requests for containerized test runs in CI, rather than relying on defaults — under-provisioned containers can cause tests to fail for reasons entirely unrelated to real application bugs, producing confusing, misleading failures.
- Decide deliberately, per project, whether local development testing should also run containerized or stay native — there's no universally correct answer, and the decision should weigh the team's actual pain points (environment drift issues vs. iteration speed needs) rather than defaulting either way without consideration.

## Common Pitfalls

- Assuming a test failure that only reproduces in a container is "just flaky" without investigating whether it's actually a genuine, categorizable environment difference (resource limits, networking, headless-specific behavior).
- Hardcoding `localhost` in test configuration, breaking when the same code needs to run inside a Compose stack where services communicate via service names instead.
- Containerizing local development testing by default without considering the iteration-speed cost, when the team's actual pain point (if any) might be entirely CI-specific and not require local containerization at all.
- Not adjusting timeouts/resource expectations for containerized execution, assuming timing behavior tuned for a full local machine transfers directly to a more resource-constrained CI container.

## Interview Notes

- Be ready to give a balanced answer on when containerizing test execution helps versus when it's unnecessary overhead — avoiding a blanket "always containerize" answer shows real, considered judgment.
- Understand the specific categories of "works locally, fails in container" issues (headless behavior, resource constraints, networking, file paths) — being able to name and diagnose these specifically is more valuable than a vague "environment differences" answer.
- Be able to describe how you'd investigate a test that only fails in a containerized environment — checking resource limits and networking assumptions first, rather than immediately assuming flakiness.

## References

- [Docker — Runtime Options for CPU and Memory](https://docs.docker.com/engine/containers/resource_constraints/)
- [Playwright — Docker](https://playwright.dev/docs/docker)