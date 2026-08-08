# Levels of Testing

## What It Is

Levels of testing are the stages at which software is tested, organized by scope — from a single function to the entire system running in a production-like environment. The four classic levels are:

1. **Unit Testing** — tests a single function, method, or class in isolation.
2. **Integration Testing** — tests how multiple units/modules work together (e.g., service ↔ database, service ↔ service).
3. **System Testing** — tests the complete, integrated application against requirements, end-to-end.
4. **Acceptance Testing** — validates the system against business needs, often performed by or with stakeholders (UAT — User Acceptance Testing).

This is distinct from **types** of testing (functional, performance, security, usability) — levels describe *scope*, types describe *what aspect* is being checked. A single level (e.g., system testing) can include multiple types (functional + performance).

## Why It Matters

- Each level catches different classes of defects. Unit tests catch logic errors cheaply and fast; system tests catch integration and environment issues that unit tests structurally cannot see.
- As an SDET, you're expected to know *which level* a given automated check belongs to, because it determines: where it lives in the codebase, how fast it should run, who owns it (dev vs QA), and how often it runs in CI.
- Misplacing tests at the wrong level (e.g., testing business logic edge cases only through slow UI end-to-end tests) is one of the most common causes of slow, flaky test suites.

## How It Works

These levels map to the **Testing Pyramid**, a strategy model for how much automated coverage should exist at each level:

```text
        /\
       /UI\        <- few, slow, expensive, high confidence per test
      /------\
     / API/   \     <- more, faster, focused on integration
    /Integr.   \
   /--------------\
  /   Unit Tests    \  <- many, fast, cheap, isolated
 /--------------------\
```

- **Unit tests**: written by developers, run on every commit, milliseconds each, no external dependencies (DB/network are mocked).
- **Integration tests**: verify real interactions between components — e.g., does the service correctly write to the actual database schema.
- **System/E2E tests**: exercise real user flows through the full stack (UI or API), closest to how a user experiences the app, but slowest and most fragile.
- **Acceptance tests**: confirm the feature satisfies the business requirement — can be manual (UAT) or automated (BDD-style, e.g., Gherkin/Cucumber).

## Example

The same requirement — "apply a 10% discount for orders over ₹1000" — tested at different levels:

```python
# UNIT TEST — tests the pure logic in isolation, no DB/HTTP involved
def calculate_discount(order_total: float) -> float:
    return order_total * 0.10 if order_total > 1000 else 0

def test_discount_applied_above_threshold():
    assert calculate_discount(1500) == 150

def test_no_discount_at_or_below_threshold():
    assert calculate_discount(1000) == 0
```

```python
# API / INTEGRATION TEST — verifies the real endpoint applies the logic correctly
import requests

def test_order_api_applies_discount():
    response = requests.post(
        "https://api.example.com/orders",
        json={"items": [{"price": 1500, "qty": 1}]}
    )
    assert response.status_code == 201
    assert response.json()["discount"] == 150
```

```python
# E2E / SYSTEM TEST (Playwright) — verifies the discount is visible to a real user in the UI
from playwright.sync_api import Page

def test_discount_shown_at_checkout(page: Page):
    page.goto("https://shop.example.com/cart")
    page.get_by_test_id("add-item-1500").click()
    page.goto("https://shop.example.com/checkout")
    assert page.get_by_test_id("discount-amount").inner_text() == "₹150"
```

## Production Considerations

- The pyramid shape is a guideline, not a law — API-heavy systems often favor a "trophy" or "diamond" shape with more integration/API tests than unit tests, since integration bugs are more common than pure logic bugs in service-oriented systems.
- As an SDET, part of your job is pushing coverage *down* the pyramid where possible — if a bug can be caught by a fast unit or API test, it shouldn't rely on a slow UI test.
- CI pipelines typically run levels in stages: unit tests on every push (seconds), integration/API tests on every PR (minutes), full E2E suites on a schedule or pre-release (longer, since they're expensive).

## Common Pitfalls

- **Ice-cream cone anti-pattern**: too many slow, flaky UI tests and too few unit tests — common in QA-led automation efforts that skip developer buy-in.
- Duplicating the same logic check across all four levels instead of testing it once at the appropriate (usually lowest) level and only checking integration/flow at higher levels.
- Treating "system testing" and "E2E testing" as scope-limited to the UI only — system testing can and should include API-level system tests too.
- Confusing "levels" with "types" in interviews — scope vs. aspect are different axes.

## Interview Notes

- Be ready to draw and explain the testing pyramid, and to justify *why* it's shaped that way (cost, speed, feedback loop).
- Know how to argue where a given test case should live and why — this is a common practical interview question ("would you automate this at the unit, API, or UI level, and why?").
- Understand that levels are owned by different people in shift-left teams: developers usually own unit/integration, SDETs often own API/E2E frameworks, and product/business owns UAT.

## References

- [ISTQB Foundation Level Syllabus — Test Levels](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)