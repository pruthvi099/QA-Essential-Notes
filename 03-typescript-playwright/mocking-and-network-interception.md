# Mocking & Network Interception

## What It Is

Playwright can intercept, inspect, and modify network requests/responses at the browser level using `page.route()` — this enables mocking API responses (testing UI behavior against controlled, synthetic data), simulating errors/slow networks, and blocking unnecessary requests (analytics, ads) to speed up tests. This is a TypeScript-focused deep dive; the same `route()` API exists in Python, but network mocking patterns are especially common in TypeScript frontend-heavy frameworks, which is why it's covered here in depth.

## Why It Matters

- Testing how the UI handles specific backend states — an error response, an empty list, a slow network — is often impractical to trigger reliably against a real backend, but trivial to simulate by intercepting the network request and returning a controlled fake response.
- Network mocking decouples frontend UI tests from backend availability and state entirely — a UI test verifying "shows an error banner on a 500 response" doesn't need a real backend to actually return a 500, which is both faster and far more reliable than trying to force a real error condition.
- This is an increasingly expected skill as frontend testing strategies mature — interviewers ask about mocking specifically to see if a candidate can test edge cases (errors, empty states, slow responses) that are hard or risky to reproduce against a live system.

## How It Works

`page.route(urlPattern, handler)` intercepts any request matching the URL pattern before it reaches the network, letting you:
- **Fulfill** — return a custom, fake response without ever hitting the real server (`route.fulfill()`)
- **Abort** — block the request entirely (`route.abort()`) — useful for blocking third-party scripts/analytics
- **Continue** — let the real request proceed, optionally after modifying it (`route.continue()`)
- **Inspect** — read the real response for verification without altering it (combined with `page.on('response')` or `route.fetch()`)

## Example

Mocking an API error response to test the UI's error-handling behavior — a state that would be difficult to reliably reproduce against a real backend:

```typescript
import { test, expect } from '@playwright/test';

test('shows error banner when orders API fails', async ({ page }) => {
  // Intercept the real request and return a fake 500 response instead
  await page.route('**/api/orders', async (route) => {
    await route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal Server Error' }),
    });
  });

  await page.goto('/account/orders');

  await expect(page.getByText('Something went wrong loading your orders')).toBeVisible();
});
```

Mocking a slow network to test loading states:
```typescript
test('shows loading spinner during slow request', async ({ page }) => {
  await page.route('**/api/orders', async (route) => {
    await new Promise(resolve => setTimeout(resolve, 2000));  // simulate delay
    await route.continue();   // then let the real request proceed
  });

  await page.goto('/account/orders');
  await expect(page.getByTestId('loading-spinner')).toBeVisible();
});
```

Mocking an empty list response to test an empty state:
```typescript
test('shows empty state when there are no orders', async ({ page }) => {
  await page.route('**/api/orders', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([]),
    });
  });

  await page.goto('/account/orders');
  await expect(page.getByText('You have no orders yet')).toBeVisible();
});
```

Blocking unnecessary third-party requests to speed up a test run:
```typescript
test('checkout flow without analytics overhead', async ({ page }) => {
  await page.route('**/*analytics*', route => route.abort());
  await page.route('**/*doubleclick*', route => route.abort());

  await page.goto('/checkout');
  // ... rest of the test, now without waiting on unrelated third-party calls
});
```

## Production Considerations

- Network mocking is best used for edge cases (errors, empty states, slow responses) that are hard to reliably reproduce against a real backend — for normal happy-path flows, testing against a real (or realistic staging) API gives higher confidence that the actual integration works, not just that the UI handles a fake response correctly.
- Keep mocked response shapes in sync with the real API contract — a mock that no longer matches what the backend actually returns will pass tests while masking a real integration break; typing mock response bodies against the same interfaces used elsewhere (see [Interfaces & Type-Safe Test Data](./interfaces-and-type-safe-test-data.md)) helps catch this drift at compile time.
- Blocking third-party requests (analytics, ads) to speed up test runs is a reasonable, low-risk optimization — but don't block requests that are actually part of what's being tested (e.g., don't block the payment provider's script in a checkout test).

## Common Pitfalls

- Over-relying on mocked responses for core, high-value flows instead of testing against a real backend at least somewhere in the suite — an entirely-mocked test suite can pass while the real integration is actually broken.
- Letting mocked response shapes drift out of sync with the real API over time, especially after a backend change — untyped, hand-written mock bodies are especially prone to this compared to ones built from shared interfaces.
- Using overly broad URL patterns in `page.route()` that accidentally intercept unrelated requests, causing confusing failures elsewhere in the same test.
- Forgetting that a mocked route persists for the rest of the test (or until unrouted) — a route set up for one specific check earlier in a test can silently affect a later, unrelated request in the same test if not scoped or cleared appropriately.

## Interview Notes

- Be ready to explain when network mocking is the right tool (testing error/empty/slow states) versus when testing against a real backend is more appropriate (core happy-path integration confidence) — this trade-off is the crux of most mocking-related interview questions.
- Understand the difference between `route.fulfill()` (fake response, never hits the real server), `route.abort()` (blocks the request), and `route.continue()` (lets it proceed, optionally modified).
- Be able to describe a specific edge case (error state, empty state, slow network) you'd test using mocking and explain why it would be impractical to reliably reproduce against a real backend.

## References

- [Playwright — Network (Node.js)](https://playwright.dev/docs/network)
- [Playwright — Mock APIs (Node.js)](https://playwright.dev/docs/mock)