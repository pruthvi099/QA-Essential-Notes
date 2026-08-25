# Test Framework Architecture Fundamentals

## What It Is

A test framework is more than a collection of test files — it's a structured system with distinct layers, each with a single responsibility: configuration, reusable utilities, page/component objects, test data, and the tests themselves. This note establishes the layered architecture model that every other note in this folder builds on, synthesizing patterns introduced individually throughout the repo (POM, fixtures, config) into one coherent architectural picture.

## Why It Matters

- A framework without clear layer separation becomes unmaintainable as it grows — logic gets duplicated, changes ripple unpredictably across unrelated tests, and onboarding new contributors becomes slow because there's no consistent place to look for a given kind of code.
- This is the conceptual foundation for [QA Engineer vs. SDET](../00-start-here/qa-engineer-vs-sdet.md)'s distinction — building and maintaining this layered architecture *is* the "framework ownership" that differentiates SDET-level work from writing individual tests.
- Interviewers frequently ask "how would you structure a test framework from scratch" specifically to assess whether a candidate thinks architecturally, not just about individual test-writing techniques covered elsewhere in this repo.

## How It Works

**The standard layers, from lowest to highest level:**

1. **Configuration layer** — environment settings, base URLs, timeouts (see [Hooks & Configuration](../02-automation-python-playwright/hooks-and-config.md), [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md)).
2. **Driver/tool abstraction layer** — wraps the underlying automation tool (Playwright, Appium) so the rest of the framework doesn't depend directly on its specific API (see [Abstraction Layers & Driver Wrappers](./abstraction-layers-and-driver-wrappers.md)).
3. **Utility/helper layer** — genuinely reusable functions not tied to any specific page (date formatting, API client wrappers, data generators).
4. **Page/component object layer** — encapsulates UI interaction (see [Page Object Model](../02-automation-python-playwright/page-object-model.md), [Designing a Scalable Page Object Hierarchy](./designing-a-scalable-page-object-hierarchy.md)).
5. **Test data layer** — factories, fixtures, seed data (see [Interfaces & Type-Safe Test Data](../03-typescript-playwright/interfaces-and-type-safe-test-data.md)).
6. **Test layer** — the actual test files, which should read as a sequence of business actions and assertions, relying on everything below.

**The core architectural principle:** each layer should only depend on layers below it, never above — a page object shouldn't know about specific test scenarios; a utility function shouldn't know about specific pages. Violating this direction is what causes tangled, hard-to-change frameworks.

## Example

A directory structure embodying this layered architecture, extending the pattern from [Modules & Project Structure](../03-typescript-playwright/modules-and-project-structure.md) with explicit layer labeling:

```text
test-framework/
├── config/                    # LAYER 1: configuration
│   ├── environments.ts
│   └── playwright.config.ts
├── core/                      # LAYER 2: driver/tool abstraction
│   └── browser-driver.ts
├── utils/                     # LAYER 3: reusable utilities
│   ├── api-client.ts
│   └── date-helpers.ts
├── pages/                     # LAYER 4: page/component objects
│   ├── BasePage.ts
│   ├── LoginPage.ts
│   └── components/
│       └── NavBar.ts
├── factories/                 # LAYER 5: test data
│   └── order-factory.ts
└── tests/                     # LAYER 6: tests
    └── checkout.spec.ts
```

A concrete illustration of the dependency direction principle — what's allowed versus what breaks the architecture:

```typescript
// ALLOWED: a page object (layer 4) depends on a utility (layer 3)
// pages/CheckoutPage.ts
import { formatCurrency } from '../utils/date-helpers';

export class CheckoutPage {
  async getDisplayedTotal(): Promise<string> {
    // uses a lower-layer utility — fine, respects the dependency direction
    return formatCurrency(await this.totalLocator.innerText());
  }
}
```

```typescript
// VIOLATION: a utility (layer 3) depending on a page object (layer 4) —
// this inverts the intended dependency direction
// utils/date-helpers.ts
import { CheckoutPage } from '../pages/CheckoutPage';   // WRONG — utilities
                                                          // should never import
                                                          // from page objects

export function formatCurrency(page: CheckoutPage) {
  // A utility this specific to one page isn't a genuine utility —
  // it should just be a method ON CheckoutPage instead
}
```

## Production Considerations

- Introduce layered structure early, even for a small framework — retrofitting clear separation onto a tangled, flat pile of test files later is significantly more disruptive than starting with the structure from day one (the same principle noted in [Modules & Project Structure](../03-typescript-playwright/modules-and-project-structure.md)).
- Not every framework needs all six layers fully fleshed out from the start — a small project might combine utility and driver-abstraction layers initially, and split them later as genuine need emerges; the principle (clear separation, correct dependency direction) matters more than rigidly implementing every layer immediately.
- Layer violations tend to creep in gradually under time pressure ("I'll just quickly reference this page object from a utility function") — periodic architecture review (or lint rules restricting cross-layer imports) helps catch this drift before it compounds.

## Common Pitfalls

- Mixing test logic directly into page objects (assertions specific to one test scenario living inside a reusable page class) — this couples a reusable component to one specific use case, reducing its reusability for other tests.
- Violating the dependency direction (a lower layer depending on a higher one), which creates circular or tangled dependencies that make the codebase confusing to navigate and risky to change.
- Treating "framework architecture" as a one-time setup task rather than an ongoing discipline — layer violations and duplicated logic accumulate gradually if not actively watched for during code review (see [Code Review for Test Code](../08-git-github/code-review-for-test-code.md)).
- Over-architecting a small framework with all six layers fully separated from day one when the project doesn't yet have enough complexity to justify it — appropriate architecture scales with actual need, not with theoretical completeness.

## Interview Notes

- Be ready to sketch a layered test framework architecture from scratch, including the dependency direction principle — one of the most common SDET system-design-style interview questions.
- Understand why page objects shouldn't contain test-specific assertions, and be able to explain the reusability cost of violating this.
- Be able to describe how you'd recognize and fix a layer-violation "smell" in an existing codebase — this shows practical architectural judgment, not just theoretical layering knowledge.

## References

- [Playwright — Page Object Models (Node.js)](https://playwright.dev/docs/pom)
- [Martin Fowler — PageObject](https://martinfowler.com/bliki/PageObject.html)