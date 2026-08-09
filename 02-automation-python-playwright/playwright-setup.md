# Playwright Setup (Python & TypeScript)

## What It Is

Playwright is a browser automation framework from Microsoft that supports Chromium, Firefox, and WebKit through a single API. This note covers installing and running your first test using both official test runners: **Pytest** (Python) and **Playwright Test** (TypeScript) — the two officially supported, first-class ways to write and run Playwright tests.

## Why It Matters

- Setup is the foundation every later note builds on — locators, fixtures, and CI integration all assume a correctly configured project.
- Knowing both ecosystems matters practically: many teams standardize on TypeScript for its native alignment with Playwright Test (built by the same team), while Python shops often prefer it for existing pytest infrastructure or data/backend tooling overlap.
- Interviewers commonly ask "walk me through setting up a Playwright project" as an opening technical question — being fluent in either stack's setup signals real hands-on experience, not just familiarity with the API.

## How It Works

**Python** uses `pytest` as the test runner, with the `pytest-playwright` plugin providing fixtures (`page`, `browser`, `context`) automatically.

**TypeScript** uses `@playwright/test`, Playwright's own purpose-built test runner — it ships built-in parallelism, retries, tracing, and fixtures without needing a separate plugin.

Both install browser binaries separately from the language package, since Playwright ships its own patched builds of Chromium/Firefox/WebKit for consistency across environments.

## Example

**Python setup:**
```bash
pip install pytest-playwright
playwright install          # downloads browser binaries
```

```python
# test_homepage.py
from playwright.sync_api import Page, expect

def test_homepage_has_title(page: Page):
    page.goto("https://playwright.dev")
    expect(page).to_have_title("Fast and reliable end-to-end testing for modern web apps | Playwright")
```

```bash
pytest test_homepage.py --headed   # run with a visible browser
```

**TypeScript setup:**
```bash
npm init playwright@latest    # scaffolds project, installs browsers, adds config
```

```typescript
// tests/homepage.spec.ts
import { test, expect } from '@playwright/test';

test('homepage has title', async ({ page }) => {
  await page.goto('https://playwright.dev');
  await expect(page).toHaveTitle(/Playwright/);
});
```

```bash
npx playwright test --headed
```

**Key structural difference:** Python tests are plain functions using a `page` fixture injected by pytest-playwright; TypeScript tests are defined via `test()` with an async callback receiving fixtures as a destructured object (`{ page }`) from `@playwright/test` directly — no separate plugin needed.

## Production Considerations

- Pin the Playwright version in both ecosystems (`requirements.txt` / `package.json`) — browser binaries are tightly coupled to the library version, and mismatches cause hard-to-diagnose failures in CI.
- `npm init playwright@latest` scaffolds a working config, example tests, and a GitHub Actions workflow automatically — Python setup requires assembling these pieces manually (`conftest.py`, `pytest.ini`, CI YAML), which is a real day-one productivity difference worth knowing about when choosing a stack.
- Install only the browsers you need in CI (`playwright install chromium`) rather than all three, to keep pipeline install times down.

## Common Pitfalls

- Forgetting to run `playwright install` after installing the package — the Python/npm package alone doesn't include browser binaries, causing a clear but easily-missed error on first run.
- Running Python tests with plain `pytest` instead of ensuring `pytest-playwright` is installed — without it, the `page` fixture doesn't exist and tests fail with a fixture-not-found error.
- Not pinning browser/library versions, leading to "works on my machine" failures when CI pulls a newer Playwright version with different binary builds.

## Interview Notes

- Be ready to explain the fixture injection difference: pytest-playwright provides `page` via a plugin; `@playwright/test` provides it as a built-in first-class fixture — this reflects that Playwright Test was purpose-built by the Playwright team specifically for this API.
- Know that `playwright install` is a separate, required step in both ecosystems — a common "gotcha" question.
- Be able to state one practical reason to choose each stack (Python: existing pytest/backend tooling; TypeScript: tightest native integration with Playwright Test's own runner features).

## References

- [Playwright — Getting Started (Python)](https://playwright.dev/python/docs/intro)
- [Playwright — Getting Started (Node.js/TypeScript)](https://playwright.dev/docs/intro)