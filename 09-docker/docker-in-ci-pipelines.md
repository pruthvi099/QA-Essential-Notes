# Docker in CI Pipelines

## What It Is

This note covers using containers within CI pipelines specifically — running test suites inside a Docker container as a CI job step (GitHub Actions, Jenkins), rather than installing dependencies directly on the CI runner. This connects [Docker Fundamentals for QA](./docker-fundamentals-for-qa.md) and [Containerizing Test Environments](./containerizing-test-environments.md) with the pipeline mechanics from [07-ci-cd](../07-ci-cd/), closing the loop on why containerization and CI consistency are often discussed together.

## Why It Matters

- Running tests in a container within CI guarantees the exact same environment every single run, regardless of what the underlying CI runner image happens to have installed or how it might drift over time — this is a stronger consistency guarantee than relying on the runner's native tooling.
- It also means the *exact same* container image used in CI can be run locally by a developer investigating a CI failure — genuinely reproducing the CI environment locally, rather than approximating it, closing the gap discussed in [Running Tests in Containers vs. Locally](./running-tests-in-containers-vs-locally.md).
- Both GitHub Actions and Jenkins have first-class support for container-based execution, making this a practical, commonly available pattern rather than an exotic setup.

## How It Works

**GitHub Actions** — supports running an entire job inside a container via the `container:` key, or running specific steps inside a container using `docker run` directly within a `run` step.

**Jenkins** — supports running a pipeline's `agent` as a Docker container (`agent { docker { image '...' } }`), giving the whole pipeline (or a specific stage) the container's environment.

## Example

**GitHub Actions — running an entire job inside the project's own test container:**
```yaml
name: Containerized E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: mcr.microsoft.com/playwright:v1.48.0-jammy   # same image used locally

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright test
      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

**GitHub Actions — building and running the project's own custom Dockerfile (from [Containerizing Test Environments](./containerizing-test-environments.md)) as a CI step:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build test image
        run: docker build -t qa-suite:ci -f tests/Dockerfile ./tests

      - name: Run tests in container
        run: |
          docker run --rm \
            -v ${{ github.workspace }}/playwright-report:/app/playwright-report \
            -e BASE_URL=${{ secrets.STAGING_BASE_URL }} \
            qa-suite:ci

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

**Jenkins — running an entire pipeline stage inside a container:**
```groovy
pipeline {
    agent none
    stages {
        stage('Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.48.0-jammy'
                }
            }
            steps {
                sh 'npm ci'
                sh 'npx playwright test'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
                }
            }
        }
    }
}
```

## Production Considerations

- Use the exact same image (or Dockerfile) both in CI and for local reproduction of CI failures — the whole point of this pattern is eliminating the gap between "what ran in CI" and "what I can reproduce locally"; using a different image for each undermines that benefit.
- Mount volumes for test report/artifact output (as shown with `-v` in the second GitHub Actions example) so results generated inside the container are accessible to the CI platform's artifact upload step — without this, reports generated inside the container are lost when it exits, the same principle as [Test Artifacts & Reports in CI](../07-ci-cd/test-artifacts-and-reports-in-ci.md) applied at the container boundary.
- Weigh container build time against the consistency benefit — if building the test image itself takes several minutes on every CI run, using a pre-built, cached image (pushed to a registry) rather than rebuilding from scratch each time can meaningfully speed up the pipeline.

## Common Pitfalls

- Using a different container image (or version) in CI than what's used for local reproduction, silently reintroducing the exact environment-drift problem containerization is meant to solve.
- Forgetting to mount/expose the report output directory from inside the container to the host CI runner, causing generated reports to be inaccessible for artifact upload — the container-specific version of forgetting `if: always()` on upload steps.
- Rebuilding the test image from scratch on every single CI run without any caching strategy, adding unnecessary time to every pipeline execution when a pre-built or cached image would suffice.
- Not accounting for container resource limits within the CI environment (see [Running Tests in Containers vs. Locally](./running-tests-in-containers-vs-locally.md)) when tests behave differently or time out inside CI's containerized execution specifically.

## Interview Notes

- Be ready to explain the practical benefit of running the *same* container image in CI and locally — specifically, that it makes CI failures genuinely, exactly reproducible on a developer's machine, not just approximately.
- Understand both patterns — running a whole CI job inside a container (GitHub Actions `container:`, Jenkins `agent { docker {...} }`) versus explicitly building/running a project's own Dockerfile as a step — and when each is appropriate.
- Be able to describe how you'd get test artifacts (reports, traces) out of a container and into the CI platform's artifact system — connecting this note's container-specific mechanics to the broader artifact strategy from [07-ci-cd](../07-ci-cd/).

## References

- [GitHub Actions — Running Jobs in a Container](https://docs.github.com/en/actions/using-jobs/running-jobs-in-a-container)
- [Jenkins — Using Docker with Pipeline](https://www.jenkins.io/doc/book/pipeline/docker/)