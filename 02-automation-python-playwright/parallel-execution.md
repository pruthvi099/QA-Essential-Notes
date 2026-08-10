# Parallel Execution (Python & TypeScript)

## What It Is

Parallel execution runs multiple tests simultaneously across multiple worker processes, instead of sequentially one after another — the single biggest lever for reducing suite runtime as a test suite grows. Both ecosystems support it, but with different tools:

- **Python** — parallelism comes from `pytest-xdist`, a separate plugin that distributes tests across multiple worker processes.
- **TypeScript** — parallelism is built into Playwright Test natively; no separate plugin needed.

## Why It Matters

- As a suite grows from 20 tests to 500+, sequential execution becomes the bottleneck between a code change and CI feedback — parallel execution is what keeps feedback loops fast at scale.
- Parallelism introduces its own class of bugs (shared state between workers, test order dependencies) that only surface once parallel execution is turned on — understanding this is essential to writing tests that are parallel-safe from the start, not retrofitted later.
- SDET interviews frequently ask how you'd speed up a slow CI suite — parallel execution (combined with proper test isolation, see [Browser Contexts & Test Isolation](./browser-contexts-and-test-isolation.md)) is the expected first answer.

## How It Works

**Python (`pytest-xdist`):**
```bash
pip install pytest-xdist
pytest -n 4          # run with 4 parallel workers
pytest -n auto        # auto-detect CPU count
```
- Distributes tests across worker processes; each worker gets its own set of test functions.
- Because each test already gets its own isolated context/page (see [Browser Contexts](./browser-contexts-and-test-isolation.md)), parallel workers don't share browser state — but they CAN share external resources (a database, a test account) unless test data is also isolated per worker.

**TypeScript (built into Playwright Test):**
```typescript
// playwright.config.ts
export default defineConfig({
  workers: 4,   // or leave unset to auto-detect based on CPU cores
  fullyParallel: true,   // run tests within a single file in parallel too, not just across files
});
```
- `workers` controls how many parallel worker processes run tests.
- `fullyParallel: true` allows tests *within the same file* to run in parallel too (by default, tests within one file run sequentially relative to each other, though files run in parallel across workers).

## Example

**Python — a parallel-unsafe test (shared resource, no isolation):**
```python
# BAD: multiple workers writing to the same hardcoded test account
# will race and produce inconsistent, order-dependent results
def test_update_profile_name():
    login_as("shared_test_user@example.com")
    update_profile_name("New Name")
    assert get_profile_name() == "New Name"
```

**Python — made parallel-safe with per-test unique data:**
```python
import uuid

def test_update_profile_name(page):
    unique_email = f"test_{uuid.uuid4().hex[:8]}@example.com"
    create_test_account(unique_email)   # each worker gets its own account
    login_as(page, unique_email)
    update_profile_name(page, "New Name")
    assert get_profile_name(page) == "New Name"
```

**TypeScript — the same principle:**
```typescript
import { test, expect } from '@playwright/test';
import { randomUUID } from 'crypto';

test('update profile name', async ({ page }) => {
  const uniqueEmail = `test_${randomUUID().slice(0, 8)}@example.com`;
  await createTestAccount(uniqueEmail);   // isolated per test/worker
  await loginAs(page, uniqueEmail);
  await updateProfileName(page, 'New Name');
  await expect(page.getByTestId('profile-name')).toHaveText('New Name');
});
```

## Production Considerations

- Parallel worker count in CI should generally match available CPU cores, not be set arbitrarily high — over-provisioning workers beyond actual resources causes resource contention and can make the suite slower, not faster.
- Any shared external resource (database, third-party test account, rate-limited API) needs a per-worker or per-test isolation strategy — otherwise parallelism surfaces flaky, order-dependent failures that don't reproduce when run sequentially, which is one of the more frustrating debugging scenarios in test automation.
- Sharding across multiple CI machines (not just multiple workers on one machine) is the next scaling step once a single machine's worker count is maxed out — both `pytest-xdist` and Playwright Test support this (`--dist` / `--shard`).

## Common Pitfalls

- Enabling parallel execution without first ensuring test data isolation, surfacing intermittent failures that are blamed on "flaky tests" when the real cause is shared state between workers.
- Assuming tests that pass sequentially will automatically pass in parallel — order-dependent tests (one test relying on state left behind by another) only break once execution order becomes non-deterministic under parallelism.
- Setting an excessively high worker count without matching CI resources, causing resource starvation (CPU/memory contention) that makes the suite slower overall.
- Not distinguishing "workers" (parallel processes) from "sharding" (splitting the suite across multiple machines) — these solve different scaling bottlenecks and are often confused in interviews.

## Interview Notes

- Be ready to explain how you'd speed up a slow CI suite, with parallel execution (and prerequisite test isolation) as the first-line answer.
- Understand the tooling difference: `pytest-xdist` (separate plugin, Python) vs. Playwright Test's native `workers`/`fullyParallel` config (TypeScript, no plugin needed).
- Be able to describe what makes a test "parallel-safe" (isolated data, no reliance on execution order, no shared mutable external resources) — this is the conceptual core interviewers are really checking for.

## References

- [pytest-xdist Documentation](https://pytest-xdist.readthedocs.io/)
- [Playwright — Parallelism and Sharding (Node.js)](https://playwright.dev/docs/test-parallel)