# 09 — Docker

Containerized test environments — from fundamentals through Compose-based integration testing, CI usage, and debugging failures inside containers. Read [02-automation-python-playwright](../02-automation-python-playwright/) first for the test suites this folder containerizes.

## Notes

1. [Docker Fundamentals for QA](./docker-fundamentals-for-qa.md) — Images, containers, and Dockerfiles
2. [Containerizing Test Environments](./containerizing-test-environments.md) — Writing an efficient Dockerfile for a Playwright suite
3. [Docker Compose for Test Stacks](./docker-compose-for-test-stacks.md) — App + database + test runner, coordinated together
4. [Running Tests in Containers vs. Locally](./running-tests-in-containers-vs-locally.md) — When containerizing helps, and common gotchas
5. [Docker in CI Pipelines](./docker-in-ci-pipelines.md) — Consistent environments in GitHub Actions and Jenkins
6. [Test Data & Volumes in Docker](./test-data-and-volumes-in-docker.md) — Persisting data vs. guaranteeing clean state
7. [Debugging Containerized Test Failures](./debugging-containerized-test-failures.md) — Logs, shells, and artifact extraction

## Related

- [07 — CI/CD](../07-ci-cd/) — the pipelines this folder's containers run inside
- [05 — SQL & Database Testing](../05-sql-database-testing/) — seeding and migration concepts used in Compose stacks