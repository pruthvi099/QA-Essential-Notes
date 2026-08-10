# Hooks & Configuration (Python & TypeScript)

## What It Is

Hooks are functions that run at defined points in the test lifecycle (before/after each test, before/after all tests in a file). Configuration defines project-wide settings (base URL, timeouts, browsers, reporters) so they're not repeated per test. Both concepts exist in Python (pytest hooks + `pytest.ini`/`conftest.py`) and TypeScript (Playwright Test hooks + `playwright.config.ts`), with different mechanisms but the same underlying goals.

## Why It Matters

- Hardcoding base URLs, timeouts, or repeated setup logic per test file makes environment switching (staging vs. production, local vs. CI) and framework-wide changes painful — centralized config and hooks solve this at the framework level, once.
- `conftest.py` (Python) and `playwright.config.ts` (TypeScript) are the first files an SDET reviewing a new codebase checks to understand how a framework is wired — knowing their role and structure is essential to onboarding onto (or building) any real framework.
- Interviewers use "how would you configure this framework for CI vs. local runs" to assess whether a candidate understands environment-driven configuration, not just individual test writing.

## How It Works

**Python — hooks via pytest:**
- `conftest.py` — a special file pytest auto-discovers; holds shared fixtures and hooks, scoped to its directory and subdirectories.
- Hooks like `pytest_configure`, `pytest_collection_modifyitems` allow customizing test collection/execution globally.
- Setup/teardown at the test level is done via fixtures (see [Fixtures](./fixtures.md)), not separate "before/after" hook functions — pytest's fixture yield pattern *is* its hook mechanism for individual tests.

**Python — configuration via `pytest.ini` / `pyproject.toml`:**
```ini
[pytest]
addopts = --browser chromium --base-url https://staging.example.com
timeout = 30
```

**TypeScript — hooks via Playwright Test:**
- `test.beforeEach()`, `test.afterEach()`, `test.beforeAll()`, `test.afterAll()` — explicit hook functions, similar to Jest/Mocha conventions.
- Hooks can be scoped per file or applied globally via a custom fixture with `{ auto: true }` (see [Fixtures](./fixtures.md)).

**TypeScript — configuration via `playwright.config.ts`:**
```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  timeout: 30000,
  use: {
    baseURL: 'https://staging.example.com',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

## Example

**Python:**
```python
# conftest.py
import pytest

@pytest.fixture(autouse=True)
def log_test_start(request):
    print(f"\nStarting test: {request.node.name}")
    yield
    print(f"Finished test: {request.node.name}")

def pytest_configure(config):
    # Global hook: runs once before the whole test session
    print("Test session starting — verifying environment reachable")
```

```python
# test_example.py — uses base_url from pytest.ini automatically
def test_homepage(page):
    page.goto("/")   # resolves against base_url configured in pytest.ini
    assert page.title() != ""
```

**TypeScript:**
```typescript
// tests/example.spec.ts
import { test, expect } from '@playwright/test';

test.beforeEach(async ({ page }, testInfo) => {
  console.log(`Starting test: ${testInfo.title}`);
});

test.afterEach(async ({}, testInfo) => {
  console.log(`Finished test: ${testInfo.title} — status: ${testInfo.status}`);
});

test('homepage loads', async ({ page }) => {
  await page.goto('/');   // resolves against baseURL from playwright.config.ts
  await expect(page).toHaveTitle(/.+/);
});
```

## Production Considerations

- Use environment variables (not hardcoded values) for anything that changes between local/CI/staging/production — both `pytest.ini` and `playwright.config.ts` can reference env vars (`os.environ` / `process.env`), keeping secrets and environment-specific values out of version control.
- `playwright.config.ts`'s `projects` array is the standard way to run the same suite across multiple browsers/devices without duplicating test files — a Python equivalent typically requires `pytest-playwright`'s `--browser` CLI flag combined with a CI matrix strategy, since pytest has no built-in "projects" concept.
- Centralize retry and timeout configuration in config files rather than scattering overrides across individual tests — consistent defaults make flaky-test patterns easier to spot (see [Retries and Timeouts](./retries-and-timeouts.md)).

## Common Pitfalls

- Duplicating setup logic across many test files instead of centralizing it in `conftest.py` / hooks — this is exactly the maintenance burden hooks and shared fixtures exist to eliminate.
- Hardcoding base URLs directly in test files, breaking the ability to point the same suite at different environments without editing test code.
- In TypeScript, confusing `test.beforeEach` (per-test, file-scoped by default) with `test.beforeAll` (once per file/worker) — using the wrong one causes either redundant setup or unintended shared state.
- Not knowing that `conftest.py` fixtures are automatically discovered based on directory location — placing a fixture in the wrong directory silently makes it unavailable to tests that need it.

## Interview Notes

- Be ready to explain what `conftest.py` is for and how pytest's fixture discovery scoping works by directory — a common practical question for Python-based frameworks.
- Understand the four Playwright Test hooks (`beforeEach`, `afterEach`, `beforeAll`, `afterAll`) and their scope — a common practical question for TypeScript-based frameworks.
- Be able to explain how you'd structure configuration to support running the same suite against multiple environments (local, staging, CI) without duplicating test code.

## References

- [Pytest — conftest.py](https://docs.pytest.org/en/stable/reference/fixtures.html#conftest-py-sharing-fixtures-across-multiple-files)
- [Playwright — Test Configuration (Node.js)](https://playwright.dev/docs/test-configuration)
- [Playwright — pytest-playwright Configuration](https://playwright.dev/python/docs/test-runners)
