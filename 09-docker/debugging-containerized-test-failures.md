# Debugging Containerized Test Failures

## What It Is

Debugging a test failure inside a container is a meaningfully different experience than debugging locally — there's no local GUI browser to watch, container logs behave differently than local console output, and getting a shell inside a running (or just-failed) container requires specific commands most engineers don't use daily. This note covers the practical toolkit for investigating a containerized test failure without needing to reproduce it locally first.

## Why It Matters

- A failure that only reproduces in a container (see the categories in [Running Tests in Containers vs. Locally](./running-tests-in-containers-vs-locally.md)) can't always be debugged by simply running the same test locally — sometimes the investigation genuinely needs to happen *inside* the container itself.
- Not knowing how to get a shell into a container, inspect its logs, or extract artifacts from it is a real, practical gap — the concepts from [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md) only help if you can actually retrieve those artifacts from inside a container that may already be stopped/removed.
- This is a frequently underestimated but genuinely common practical skill for teams using containerized test infrastructure — interviewers ask about it to distinguish candidates who've only read about Docker from those who've actually debugged something inside one.

## How It Works

**Core debugging commands:**
- `docker exec -it <container> bash` — opens an interactive shell inside a *running* container, letting you poke around, check file state, or manually re-run a command.
- `docker logs <container>` — retrieves a container's stdout/stderr output, including after it has stopped (as long as it hasn't been removed).
- `docker cp <container>:<path> <host-path>` — copies a file/directory out of a container (running or stopped) to the host machine — the key command for extracting a trace/screenshot/report after a failure.
- `docker ps -a` — lists containers including stopped ones (useful since a failed CI container is often stopped, not removed, immediately after failure — giving a window to investigate before cleanup).

**The core investigative workflow:** find the failed/stopped container → check its logs first (fast, often enough) → if more detail is needed, extract artifacts via `docker cp` → if genuinely stuck, get an interactive shell into a similar, freshly-started container to poke around.

## Example

**Investigating a failed CI container after a test run — the full practical workflow:**
```bash
# Find the container, even if it's already stopped
docker ps -a
# CONTAINER ID   IMAGE                STATUS                    NAMES
# a1b2c3d4e5f6   qa-suite:ci          Exited (1) 2 minutes ago  gallant_hopper

# Check its logs first — often enough to understand what happened
docker logs a1b2c3d4e5f6

# Extract the Playwright HTML report and traces for deeper investigation,
# even though the container itself has already stopped
docker cp a1b2c3d4e5f6:/app/playwright-report ./local-report
docker cp a1b2c3d4e5f6:/app/test-results ./local-test-results

# Open the extracted report locally, same as any other Playwright report
npx playwright show-report ./local-report
```

**Getting an interactive shell into a fresh container to investigate an environment-specific issue directly:**
```bash
# Start a NEW container from the same image, but override the default
# command with an interactive shell instead of running tests
docker run -it --rm qa-suite:ci bash

# Inside the container, manually investigate:
node --version                          # confirm expected runtime version
npx playwright --version                # confirm expected Playwright version
env | grep BASE_URL                     # check environment variables are set correctly
npx playwright test tests/checkout.spec.ts --headed=false --debug   # try re-running manually
```

**A common CI-specific technique** — deliberately keeping a failed container alive momentarily for investigation, since CI environments often clean up containers quickly:
```yaml
# GitHub Actions — a debugging step that pauses on failure, letting you
# SSH into the runner (via a debugging action) to investigate the
# actual failed container state before it's cleaned up
- name: Debug on failure
  if: failure()
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
```

## Production Considerations

- Always extract artifacts (reports, traces, logs) via `docker cp` or a mounted volume *before* a container is removed — CI environments typically clean up containers automatically after a job completes, so this extraction needs to happen as an explicit pipeline step (see [Docker in CI Pipelines](./docker-in-ci-pipelines.md)), not as an afterthought during manual investigation.
- For genuinely hard-to-reproduce containerized failures, running the exact same image interactively (`docker run -it`) and manually stepping through the failing scenario is often faster than trying to add more logging and re-running the full CI pipeline repeatedly.
- Debugging tools like `action-tmate` (interactive SSH access to a CI runner) are powerful but should be used sparingly and deliberately — leaving such access open longer than needed, or using it routinely instead of proper artifact-based debugging, is both a security consideration and a workflow inefficiency.

## Common Pitfalls

- Not extracting artifacts before a CI container is automatically cleaned up, losing exactly the debugging information needed once the investigation actually begins.
- Trying to debug a containerized failure purely by reading code and guessing, instead of using `docker exec`/`docker run -it` to directly, interactively inspect the actual environment — this wastes significant time compared to direct investigation.
- Forgetting that `docker logs` only shows what the container has already written to stdout/stderr — a test framework that suppresses output by default may require explicit verbose/debug flags to produce anything useful in the logs.
- Confusing `docker exec` (attaching to an already-running container) with `docker run` (starting a brand-new container) — using the wrong one when a container has already stopped, since `docker exec` requires the container to still be running.

## Interview Notes

- Be ready to describe your actual workflow for debugging a test that failed inside a CI container — `docker logs` → `docker cp` artifacts → interactive shell if needed — a common, practical question distinguishing real hands-on experience.
- Understand the distinction between `docker exec` (into a running container) and `docker run -it` (starting a fresh one) for debugging purposes, and when each applies.
- Be able to explain why artifact extraction needs to be an explicit pipeline step, not an afterthought, given that CI environments typically clean up containers automatically.

## References

- [Docker — docker exec](https://docs.docker.com/reference/cli/docker/container/exec/)
- [Docker — docker cp](https://docs.docker.com/reference/cli/docker/container/cp/)