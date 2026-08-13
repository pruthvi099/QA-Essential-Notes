# Async Patterns & Promise Handling

## What It Is

Playwright's TypeScript API is entirely promise-based — nearly every method returns a `Promise` that must be awaited. This note covers the async patterns that matter specifically for test automation: correctly sequencing awaits, running independent operations in parallel with `Promise.all()`, and the failure modes that come from mishandling promises in test code.

## Why It Matters

- A missing `await` is one of the most common sources of confusing, hard-to-diagnose bugs in TypeScript Playwright suites — the code compiles fine, but actions fire out of order or a test "passes" without actually waiting for what it claimed to check.
- Some operations are genuinely independent and safe to run concurrently (e.g., fetching two unrelated API resources) — knowing when `Promise.all()` is appropriate (and when it isn't, due to shared browser/page state) is a meaningful performance and correctness skill.
- This is a foundational async/await topic tested heavily in interviews, since promise-handling mistakes are common even among developers with general JavaScript/TypeScript experience but limited Playwright-specific exposure.

## How It Works

**Sequential awaits** — the default and correct pattern for actions on the same page, since they depend on prior actions completing first:
```typescript
await page.goto('/checkout');
await page.getByLabel('Card Number').fill('4242424242424242');
await page.getByRole('button', { name: 'Pay' }).click();
```

**`Promise.all()`** — run genuinely independent async operations concurrently, useful when they don't depend on each other and don't compete for the same page/browser resource:
```typescript
const [productResponse, ordersResponse] = await Promise.all([
  request.get('/api/products/101'),
  request.get('/api/orders'),
]);
```

**A common, correct Playwright-specific pattern** — using `Promise.all()` to avoid a race between triggering an action and waiting for its resulting event (navigation, download, popup):
```typescript
const [download] = await Promise.all([
  page.waitForEvent('download'),   // start waiting BEFORE the click
  page.getByRole('button', { name: 'Download Invoice' }).click(),  // triggers it
]);
```

This pattern exists because if you awaited the click first and *then* started waiting for the download event, the download could fire and complete before the listener was even attached — `Promise.all()` starts both simultaneously, avoiding that race.

## Example

**Incorrect — missing await causes actions to fire out of order:**
```typescript
test('checkout flow - BROKEN', async ({ page }) => {
  page.goto('/checkout');   // missing await!
  await page.getByLabel('Card Number').fill('4242424242424242');
  // The fill() may execute before goto() has finished navigating,
  // failing unpredictably depending on timing
});
```

**Correct — proper sequential awaiting:**
```typescript
test('checkout flow', async ({ page }) => {
  await page.goto('/checkout');
  await page.getByLabel('Card Number').fill('4242424242424242');
  await page.getByRole('button', { name: 'Pay' }).click();
});
```

**Incorrect — the download-race pattern done wrong:**
```typescript
test('download invoice - RACY', async ({ page }) => {
  await page.getByRole('button', { name: 'Download Invoice' }).click();
  // By the time we start waiting here, the download event may have
  // already fired and been missed entirely
  const download = await page.waitForEvent('download');
});
```

**Correct — using `Promise.all()` to avoid the race:**
```typescript
test('download invoice', async ({ page }) => {
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.getByRole('button', { name: 'Download Invoice' }).click(),
  ]);
  expect(download.suggestedFilename()).toBe('invoice.pdf');
});
```

## Production Considerations

- Enable the `no-floating-promises` ESLint rule (from `@typescript-eslint`) in the framework's lint config — it catches missing `await` calls automatically at lint time, turning a subtle runtime bug class into an immediate, visible lint error before code is even committed.
- Use `Promise.all()` only for genuinely independent operations — running two actions on the *same* page concurrently (e.g., two clicks) doesn't speed anything up meaningfully and can cause unpredictable interleaving, since a single page's actions are inherently sequential from the browser's perspective.
- The click-and-wait-for-event pattern (`Promise.all([waitForEvent(...), click()])`) should be the default for navigation/download/popup events specifically — it's a Playwright idiom worth standardizing across a framework rather than reinventing per test.

## Common Pitfalls

- Forgetting `await` on a Playwright method call — the test still "runs" without a compile error, but actions can fire out of order or complete before assertions check their result, causing intermittent, confusing failures.
- Using `Promise.all()` for actions that aren't actually independent (e.g., two sequential form fills on the same page) — this doesn't provide the intended speedup and can introduce race conditions where none existed before.
- Writing the click-then-wait pattern in the wrong order (click first, then start waiting for the event) — this misses events that fire and resolve faster than the listener was attached, a subtle but common source of intermittent test failures.
- Not enabling lint rules that catch missing awaits, relying instead on manually noticing them during code review — this doesn't scale as a framework and team grow.

## Interview Notes

- Be ready to explain what happens when `await` is missing on a Playwright call, and why the code still compiles despite being broken.
- Understand the click-and-wait-for-event race condition and why `Promise.all()` (with the wait started first in the array) is the correct pattern — this is one of the more specific, practically-tested Playwright TypeScript questions.
- Be able to explain when `Promise.all()` genuinely helps (independent operations) versus when it doesn't apply (sequential actions on the same page).

## References

- [Playwright — Events (Node.js)](https://playwright.dev/docs/events)
- [MDN — Promise.all()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)