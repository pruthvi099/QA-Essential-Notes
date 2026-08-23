# Test Data & Volumes in Docker

## What It Is

Docker **volumes** persist data outside a container's own filesystem, so it survives container removal and can be shared between containers or with the host machine. This note covers using volumes deliberately for test data — both for genuine persistence needs (a database that should survive between local test runs) and for the opposite, equally important need: guaranteeing a genuinely clean, isolated state for each test run.

## Why It Matters

- Containers are ephemeral by default (see [Docker Fundamentals for QA](./docker-fundamentals-for-qa.md)) — without volumes, a database container's data disappears every time it's removed, which is sometimes exactly what you want (clean state) and sometimes a real problem (losing a large seeded dataset you don't want to regenerate every run).
- Getting this wrong in either direction causes real problems: no volume when persistence is needed means slow, repeated reseeding; a leftover volume when clean state is needed means the exact test-data leakage between runs covered in [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md), now happening at the container level instead.
- This is the concrete mechanism that makes the Compose-based integration testing from [Docker Compose for Test Stacks](./docker-compose-for-test-stacks.md) actually reliable and repeatable — volume strategy is what determines whether "tear down and start fresh" genuinely produces a clean state.

## How It Works

**Volume types:**
- **Named volumes** — Docker-managed storage, persists independently of any specific container, referenced by name, survives `docker compose down` unless explicitly removed with `-v`.
- **Bind mounts** — maps a specific host machine path into the container, useful for things like seed SQL files or mounting test report output back to the host (as seen in [Docker in CI Pipelines](./docker-in-ci-pipelines.md)).

**The two opposing goals volumes serve for testing, and how to achieve each:**
1. **Persistence across runs** (e.g., a large seeded reference dataset you don't want to regenerate every time) → use a named volume, and don't remove it between runs.
2. **Guaranteed clean state for every run** (the more common testing need) → either don't use a volume at all for that data (letting the container's filesystem reset naturally), or explicitly remove the volume (`docker compose down -v`) before/after each test run.

## Example

**A named volume for persisting database data ACROSS local development sessions (not appropriate for CI, where clean state matters more):**
```yaml
# docker-compose.dev.yml — for LOCAL development convenience only
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data   # persists across `docker compose down`
    environment:
      POSTGRES_DB: dev_orders_db

volumes:
  db_data:   # named volume, survives container removal
```

**The CI/test equivalent — deliberately NOT persisting data, ensuring a genuinely clean state every single run:**
```yaml
# docker-compose.test.yml — for CI, where clean state matters MORE than
# persistence convenience
services:
  db:
    image: postgres:16
    # NO volume here — data is intentionally ephemeral, gone when the
    # container is removed, guaranteeing a clean state every CI run
    environment:
      POSTGRES_DB: test_orders_db
```

**A bind mount for seeding data from a host file (connecting to [Test Data Seeding with SQL](../05-sql-database-testing/test-data-seeding-with-sql.md)):**
```yaml
services:
  db-seed:
    image: postgres:16
    volumes:
      - ./seed_test_data.sql:/seed.sql:ro   # :ro = read-only, seed script shouldn't be modified by the container
    entrypoint: >
      psql postgresql://test_user:test_password@db:5432/test_orders_db -f /seed.sql
```

**Explicit, deliberate cleanup ensuring no volume persists between CI runs:**
```bash
# Runs the full test stack, then explicitly removes volumes too —
# not just containers — guaranteeing the NEXT run starts from
# genuinely zero leftover state
docker compose -f docker-compose.test.yml up --abort-on-container-exit
docker compose -f docker-compose.test.yml down -v   # -v removes named volumes too
```

## Production Considerations

- Be deliberate and explicit about which Compose file/context uses volumes for persistence and which doesn't — mixing a "convenient for local dev" volume strategy into a CI test configuration (or vice versa) is a common, easy-to-miss source of environment-specific behavior differences.
- `docker compose down` alone does NOT remove named volumes by default — the `-v` flag is required, and forgetting it is a common, specific way stale data silently persists between supposedly "clean" test runs.
- For large reference datasets that are genuinely expensive to regenerate every run (and don't change often), a persisted volume with a periodic, deliberate refresh policy can be a reasonable middle ground between "always reseed" and "never refresh."

## Common Pitfalls

- Adding a database volume to a CI/test Compose configuration for "convenience," inadvertently allowing test data to leak between CI runs and reintroducing the exact isolation problems [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) warns against.
- Forgetting the `-v` flag on `docker compose down`, assuming containers being removed also removes their data — named volumes specifically persist through this unless explicitly targeted.
- Not distinguishing bind mounts (host-path-specific, useful for seed files/report output) from named volumes (Docker-managed, useful for data persistence) — using the wrong type for a given need causes confusing behavior.
- Reusing the same Compose file/volume configuration for both local development convenience and CI test isolation, when these two contexts often have genuinely opposing volume needs.

## Interview Notes

- Be ready to explain the difference between named volumes and bind mounts, and give a testing-relevant use case for each.
- Understand why CI/test environments typically want to AVOID persisting database volumes between runs (clean state), while local development often wants the OPPOSITE (persistence, for convenience) — this contrast is a common, practical interview point.
- Be able to describe the specific `docker compose down -v` gotcha (forgetting `-v` leaves volumes intact) — a small, concrete detail that reveals real hands-on Docker Compose experience.

## References

- [Docker — Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Docker Compose — Volumes Top-Level Element](https://docs.docker.com/reference/compose-file/volumes/)