# Smoke vs. Regression in CI

## What It Is

This note goes deeper into structuring a pipeline's stages specifically around smoke and regression tiers (introduced conceptually in [Types of Testing](../00-start-here/types-of-testing.md) and applied to triggers in [Pull Request Test Triggers](./pull-request-test-triggers.md)) — the concrete tagging, filtering, and stage-design mechanics that make this tiering actually work in a real pipeline, across both Python and TypeScript toolchains.

## Why It Matters

- A pipeline without deliberate smoke/regression tiering either runs everything on every trigger (slow, expensive) or runs too little (misses regressions) — this is the single most impactful structural decision in most CI pipelines' test stages.
- Tagging strategy (which tests get marked `@smoke` vs. `@regression`) is an ongoing team discipline, not a one-time setup — new tests need to be deliberately tiered as they're added, or the smoke suite silently grows into a slow, untiered pile over time.
- This is a practical extension of both [Types of Testing](../00-start-here/types-of-testing.md) (the conceptual smoke/sanity/regression distinction) and [Test Annotations & Tagging](../03-typescript-playwright/test-annotations-and-tagging.md) (Playwright's specific tagging mechanics) into full pipeline design — interviewers use it to check whether these concepts connect into an actual working strategy, not isolated facts.

## How It Works

**Defining what belongs in the smoke tier — a deliberately small, high-confidence set:**
- Critical business paths only (login, checkout, core navigation) — not every feature.
- Should complete quickly (see the time-target discussion in [Pull Request Test Triggers](./pull-request-test-triggers.md)).
- A smoke failure should mean "something is fundamentally broken," not "a minor edge case regressed."

**Defining what belongs in the regression tier:**
- Broader feature coverage, edge cases, less-trafficked flows.
- Acceptable to run less frequently (post-merge, nightly) given its slower runtime.

**Tagging mechanics (both languages, tying together earlier notes):**
- **Python (pytest markers)** — `@pytest.mark.smoke`, `@pytest.mark.regression`, filtered via `pytest -m smoke`.
- **TypeScript (Playwright Test tags)** — `@smoke`/`@regression` in test titles, filtered via `--grep`/`--grep-invert` (see [Test Annotations & Tagging](../03-typescript-playwright/test-annotations-and-tagging.md)).

## Example

**Python — pytest marker configuration and CI-stage-specific execution:**
```python
# pytest.ini
[pytest]
markers =
    smoke: critical-path tests, fast feedback, run on every PR
    regression: broader coverage, run post-merge or nightly
```

```python
import pytest

@pytest.mark.smoke
@pytest.mark.regression   # a test can belong to BOTH tiers if it's both critical and detailed
def test_login_with_valid_credentials(page):
    ...

@pytest.mark.regression   # NOT smoke — an edge case, not a critical path
def test_login_with_unicode_characters_in_password(page):
    ...
```

```bash
# CI stage-specific commands
pytest -m smoke                     # fast PR check
pytest -m "regression and not smoke"  # avoid re-running smoke tests unnecessarily
                                       # in the broader regression pass
```

**TypeScript — the equivalent structure using Playwright Test tags:**
```typescript
test('login with valid credentials @smoke @regression', async ({ page }) => {
  // ...
});

test('login with unicode characters in password @regression', async ({ page }) => {
  // ...
});
```

```bash
npx playwright test --grep @smoke
npx playwright test --grep @regression --grep-invert @smoke
```

**A concrete tiering decision table for a real feature area**, showing the judgment calls this note is really about:

```text
Checkout Feature — Tiering Decisions

test_checkout_completes_with_valid_card       → @smoke @regression
  (core revenue path — must always be in the fast tier)

test_checkout_rejects_expired_card             → @regression
  (important, but not "is the site fundamentally broken" level)

test_checkout_applies_discount_code             → @regression
  (valuable coverage, not critical-path-blocking)

test_checkout_handles_100_char_promo_code_input → @regression
  (edge case — clearly regression-only, never smoke)
```

## Production Considerations

- Review the smoke tier's size and runtime periodically — it's meant to stay small and fast; if it's grown to the point where it no longer completes quickly, some tests likely need to be reclassified as regression-only.
- A test can legitimately belong to both tiers (as `test_login_with_valid_credentials` does above) — this isn't a design flaw, it just means the test is both critical-path *and* worth including in the broader regression sweep for completeness.
- Treat tiering decisions as something made deliberately when a test is written, not retrofitted later — the author of a new test is best positioned to judge whether it covers a critical path or an edge case, and building this into the test-writing habit prevents tier drift over time.

## Common Pitfalls

- Letting the smoke tier grow unchecked as new tests default to being tagged `@smoke` "just in case," eventually making it as slow as full regression and defeating its entire purpose.
- Not tagging new tests at all, leaving them to run only in whatever the "default" untagged execution happens to be — this creates inconsistent, unpredictable coverage across pipeline stages.
- Tiering purely by feature area ("everything in checkout is smoke") instead of by actual criticality of the specific scenario — not every checkout test is equally critical, as the example table above shows.
- Never revisiting tiering decisions as the application evolves — a feature that was once minor can become business-critical over time, and its test tier should be reconsidered accordingly.

## Interview Notes

- Be ready to design a tiering strategy for a described set of features, explaining which specific tests you'd mark smoke versus regression and why — a very common, practical exercise building directly on [Types of Testing](../00-start-here/types-of-testing.md).
- Understand that a test can belong to multiple tiers, and be able to explain when that's appropriate versus when it signals unclear tiering discipline.
- Be able to describe how you'd notice and address smoke-tier scope creep over time — this shows ongoing pipeline stewardship, not just initial setup thinking.

## References

- [Pytest — Marking Test Functions](https://docs.pytest.org/en/stable/how-to/mark.html)
- [Playwright — Tag Tests (Node.js)](https://playwright.dev/docs/test-annotations#tag-tests)