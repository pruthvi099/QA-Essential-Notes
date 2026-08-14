# Playwright Component Testing

## What It Is

Playwright Component Testing (CT) lets you mount and test individual UI components (React, Vue, Svelte) in a real browser, in isolation, without needing a full running application. It sits between unit testing (pure logic, no rendering) and full E2E testing (the whole app) — you get real browser rendering and Playwright's locator/assertion API, but scoped to a single component with controlled props.

This is a genuinely TypeScript/JavaScript-only capability — Playwright Component Testing has no Python equivalent, since it's tied to JS/TS component frameworks (React, Vue, Svelte) that don't have a meaningful Python-side parallel.

## Why It Matters

- Testing a component in isolation is faster and more focused than testing it only through a full E2E flow — you can test every prop combination and edge case a component supports without needing to navigate an entire application to reach it.
- CT fills a real gap in the testing pyramid (see [Levels of Testing](../00-start-here/levels-of-testing.md)): pure unit tests (Jest/Vitest with a DOM simulator like jsdom) don't render in a real browser and can miss real rendering/CSS/interaction issues; full E2E tests are slow and don't isolate a single component's behavior well. CT gets real-browser fidelity at component-level speed and focus.
- This is a newer, less universally known Playwright capability — being able to speak to it (and when it's the right tool vs. unit or E2E testing) differentiates a candidate with current, hands-on Playwright ecosystem knowledge.

## How It Works

CT uses a separate config (`playwright-ct.config.ts`) and a `mount()` fixture that renders a component directly, with props passed in, inside a real browser context — no application routing, backend, or full page load involved.

**Setup (React example):**
```bash
npm init playwright@latest -- --ct
```

This scaffolds a `playwright-ct.config.ts` and installs the framework-specific CT package (`@playwright/experimental-ct-react`, `-vue`, or `-svelte`).

## Example

Testing a `DiscountBadge` React component in isolation, across different prop combinations:

```typescript
// DiscountBadge.tsx
interface DiscountBadgeProps {
  percentage: number;
  active: boolean;
}

export function DiscountBadge({ percentage, active }: DiscountBadgeProps) {
  if (!active) return null;
  return <span data-testid="discount-badge">{percentage}% OFF</span>;
}
```

```typescript
// DiscountBadge.spec.tsx
import { test, expect } from '@playwright/experimental-ct-react';
import { DiscountBadge } from './DiscountBadge';

test('shows discount percentage when active', async ({ mount }) => {
  const component = await mount(<DiscountBadge percentage={20} active={true} />);
  await expect(component.getByTestId('discount-badge')).toHaveText('20% OFF');
});

test('renders nothing when inactive', async ({ mount }) => {
  const component = await mount(<DiscountBadge percentage={20} active={false} />);
  await expect(component).not.toBeVisible();
});

test('handles a range of percentage values', async ({ mount }) => {
  for (const pct of [0, 10, 50, 99]) {
    const component = await mount(<DiscountBadge percentage={pct} active={true} />);
    await expect(component.getByTestId('discount-badge')).toHaveText(`${pct}% OFF`);
  }
});
```

Notice there's no `page.goto()` at all — `mount()` renders just this one component directly, with the exact prop combinations the test controls, none of which requires navigating a real app or seeding backend data.

## Production Considerations

- Use CT for components with meaningful internal logic/states (conditional rendering, prop-driven variants, interactive widgets) — a purely presentational component with no logic gets little value from CT beyond what a visual/snapshot check would already give.
- CT tests still need real backend/E2E coverage for the *integration* of a component into the full app (correct data flowing in, correct events flowing out to parent components) — CT verifies the component in isolation, not that it's wired correctly into the larger application.
- Keep CT config (`playwright-ct.config.ts`) and regular E2E config (`playwright.config.ts`) separate — they have different fixtures (`mount` vs. `page`-only) and typically different projects/browser needs.

## Common Pitfalls

- Trying to use CT as a full replacement for E2E testing — CT can't catch integration issues (wrong data passed from a parent, broken routing) since it deliberately isolates the component from the rest of the app.
- Testing components with heavy external dependencies (API calls, global state) without mocking them first — CT mounts the component in isolation, so unmocked dependencies either fail or require an actual backend, defeating the purpose of isolated testing.
- Confusing CT with unit testing tools like Jest/Vitest + Testing Library — CT specifically renders in a real browser via Playwright, giving real CSS/layout/interaction fidelity that jsdom-based unit tests don't have; they're complementary, not interchangeable.

## Interview Notes

- Be ready to explain where Component Testing fits relative to unit testing and E2E testing — a common way to check whether a candidate understands the testing pyramid practically, not just in theory.
- Understand that CT is React/Vue/Svelte-specific tooling with no Python-Playwright equivalent — a good example of a place the two ecosystems genuinely diverge, not just in syntax.
- Be able to describe a real component you'd choose to test with CT versus one you wouldn't (e.g., a stateful discount badge vs. a static footer) — this shows practical judgment about where the tool adds value.

## References

- [Playwright — Component Testing (Node.js)](https://playwright.dev/docs/test-components)