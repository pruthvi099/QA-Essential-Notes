# Docker Compose for Test Stacks

## What It Is

Docker Compose lets you define and run multiple related containers together as a single, coordinated stack — an application, a database, and a test runner, all started, networked, and torn down together via one YAML file (`docker-compose.yml`). This solves a common integration-testing need: spinning up a complete, realistic environment on demand, locally or in CI, without depending on a shared, potentially inconsistent staging environment.

## Why It Matters

- Integration tests (see [Levels of Testing](../00-start-here/levels-of-testing.md)) genuinely need the application and its dependencies (database, cache, other services) running together — Compose makes this reproducible and disposable, rather than relying on a fragile, shared staging environment that many tests/people compete for.
- This directly extends [Database Migration Testing](../05-sql-database-testing/database-migration-testing.md) and [Test Data Seeding with SQL](../05-sql-database-testing/test-data-seeding-with-sql.md) — a Compose stack is a natural, self-contained place to run migrations and seed data before tests execute, in a fully isolated environment.
- Being able to spin up a full local test stack on demand is a practical, everyday productivity tool — it removes dependency on a shared staging environment for local development and debugging, letting an SDET reproduce integration issues without needing scarce, contested shared infrastructure.

## How It Works

**Core concepts:**
- **Service** — one component of the stack (the app, the database, the test runner) — each becomes its own container.
- **Networking** — Compose automatically creates a shared network so services can reach each other by service name (e.g., the app connects to `db:5432`, not `localhost:5432`).
- **`depends_on`** — declares startup order dependencies, though this alone doesn't guarantee a dependency is actually *ready* (see the health-check pattern below).
- **Volumes** — used here for persisting database data across runs when needed (see [Test Data & Volumes in Docker](./test-data-and-volumes-in-docker.md) for a full treatment).

## Example

A complete test stack: application, database, and a Playwright test runner, all coordinated via Compose:

```yaml
# docker-compose.test.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: test_orders_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test_user"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: ./app
    environment:
      DATABASE_URL: postgresql://test_user:test_password@db:5432/test_orders_db
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy   # waits for the ACTUAL healthcheck, not just container start

  test-runner:
    build:
      context: ./tests
      dockerfile: Dockerfile
    environment:
      BASE_URL: http://app:3000
      DATABASE_URL: postgresql://test_user:test_password@db:5432/test_orders_db
    depends_on:
      app:
        condition: service_started
    command: npx playwright test
```

```bash
# Start the entire stack, run tests, and tear everything down —
# a fully self-contained, reproducible integration test run
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test-runner

# Clean up afterward (removes containers AND the network)
docker compose -f docker-compose.test.yml down -v
```

**A pre-test seeding step**, connecting this to [Test Data Seeding with SQL](../05-sql-database-testing/test-data-seeding-with-sql.md), run as part of the same stack before tests execute:
```yaml
  db-seed:
    image: postgres:16
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./seed_test_data.sql:/seed.sql
    entrypoint: >
      psql postgresql://test_user:test_password@db:5432/test_orders_db -f /seed.sql
```

## Production Considerations

- Use `condition: service_healthy` with an actual healthcheck (not just `condition: service_started`) for dependencies where "started" doesn't mean "actually ready" — a database container can report as started before it's actually accepting connections, and this gap is a common source of flaky integration test runs against a Compose stack.
- Always tear down the stack after tests (`docker compose down -v`), including volumes, to ensure the next run starts from a genuinely clean state — a leftover stack from a previous run can cause confusing, hard-to-reproduce state issues.
- Keep the test-stack Compose file separate from any development or production Compose configuration (as the `docker-compose.test.yml` naming here suggests) — mixing test-specific and dev/prod configuration in one file risks accidentally using test settings (weak passwords, debug flags) somewhere they shouldn't be.

## Common Pitfalls

- Relying on `depends_on` alone (without a healthcheck) to ensure a dependency is actually ready, causing intermittent failures where the app or test runner starts before the database can accept connections.
- Not tearing down the stack (including volumes) between test runs, leading to leftover data from a previous run silently affecting the next one — the same isolation principle from [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md), now at the container-stack level.
- Hardcoding `localhost` in service configuration instead of the Compose service name (`db`, `app`) — within a Compose network, services reach each other by service name, not `localhost`, which refers to the container itself.
- Mixing test-specific secrets/passwords (fine to hardcode in a throwaway local test stack) with real credential-handling patterns — always confirm a Compose file is genuinely test-only before assuming its hardcoded values are safe.

## Interview Notes

- Be ready to explain how Docker Compose networking lets services reach each other by service name, and why `localhost` doesn't work the same way inside a Compose stack.
- Understand the `depends_on` + healthcheck pattern and why `depends_on` alone is often insufficient for ensuring a dependency is truly ready — a common, practical gotcha interviewers ask about.
- Be able to describe a full integration-testing workflow using Compose: stack up, seed data, run tests, capture results, tear down — showing an end-to-end understanding, not just isolated Compose syntax knowledge.

## References

- [Docker Compose — Documentation](https://docs.docker.com/compose/)
- [Docker Compose — Healthchecks](https://docs.docker.com/reference/compose-file/services/#healthcheck)