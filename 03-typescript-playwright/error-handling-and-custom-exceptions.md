# Error Handling & Custom Exceptions

## What It Is

Error handling in a TypeScript test framework covers how failures — expected and unexpected — are caught, wrapped, and surfaced with useful context. This includes `try/catch` with async/await, creating custom error classes for framework-specific failures, and deliberately testing that an application handles errors correctly (negative-path testing).

## Why It Matters

- A generic error message ("Error: undefined") from deep inside a framework utility wastes debugging time — custom, descriptive errors that include relevant context (what was being attempted, with what data) turn a confusing failure into an immediately actionable one.
- Negative-path testing — verifying an application fails gracefully and correctly (proper error messages, correct status codes) — is as important as verifying the happy path, and requires understanding how to correctly write assertions around expected failures rather than accidentally letting them crash the test itself.
- Interviewers use error-handling questions to assess whether a candidate writes production-quality framework code or just happy-path test scripts — clear, well-typed error handling is a hallmark of the SDET-level code quality discussed in [QA Engineer vs. SDET](../00-start-here/qa-engineer-vs-sdet.md).

## How It Works

**Custom error classes** — extend the built-in `Error` class to create framework-specific, identifiable error types:
```typescript
class ApiError extends Error {
  constructor(public status: number, public url: string, message: string) {
    super(message);
    this.name = 'ApiError';
  }
}
```

**Try/catch with async/await:**
```typescript
try {
  await someAsyncOperation();
} catch (error) {
  // error is typed as `unknown` in strict TypeScript — narrow it before use
  if (error instanceof ApiError) {
    console.error(`API call to ${error.url} failed with status ${error.status}`);
  }
  throw error;   // re-throw unless this is the point where it should be handled
}
```

**Negative-path testing** — asserting that a failure happens correctly, using Playwright's assertions rather than a try/catch around the test itself:
```typescript
// Testing that an API correctly REJECTS invalid input
const response = await request.post('/api/orders', { data: { items: [] } });
expect(response.status()).toBe(400);
```

## Example

A typed API client that wraps failures in a custom error with useful context, and a test verifying both a success and a deliberate failure path:

```typescript
// api-client.ts
export class ApiError extends Error {
  constructor(public status: number, public url: string, public body: unknown) {
    super(`API request to ${url} failed with status ${status}`);
    this.name = 'ApiError';
  }
}

import { APIRequestContext } from '@playwright/test';

export async function postJson<T>(
  request: APIRequestContext,
  url: string,
  data: unknown
): Promise<T> {
  const response = await request.post(url, { data });

  if (!response.ok()) {
    const body = await response.json().catch(() => null);
    throw new ApiError(response.status(), url, body);
  }

  return response.json() as Promise<T>;
}
```

```typescript
// tests/order-creation.spec.ts
import { test, expect } from '@playwright/test';
import { postJson, ApiError } from '../api-client';
import { Order } from '../types';

test('creates an order successfully', async ({ request }) => {
  const order = await postJson<Order>(request, '/api/orders', {
    items: [{ productId: 101, qty: 1 }],
  });
  expect(order.id).toBeGreaterThan(0);
});

test('rejects an order with no items', async ({ request }) => {
  // Deliberately triggering the failure path, then asserting
  // it failed with the RIGHT error — not just "it threw something"
  try {
    await postJson(request, '/api/orders', { items: [] });
    throw new Error('Expected postJson to throw, but it succeeded');
  } catch (error) {
    expect(error).toBeInstanceOf(ApiError);
    if (error instanceof ApiError) {
      expect(error.status).toBe(400);
    }
  }
});
```

## Production Considerations

- Custom error classes should carry structured context (status code, URL, relevant IDs) as typed properties, not just a formatted message string — this lets calling code programmatically branch on error type/details, not just log and re-throw.
- Distinguish between errors that should fail the test immediately (an unexpected application error mid-test) and errors that are the actual subject of a negative-path test (an expected 400 response) — conflating the two either masks real bugs or makes negative-path tests unnecessarily awkward to write.
- In strict TypeScript, caught errors are typed `unknown`, not `Error` — always narrow with `instanceof` before accessing custom properties; skipping this either causes a compile error or, worse, unsafely assumes a shape the error might not actually have.

## Common Pitfalls

- Wrapping negative-path assertions in a try/catch around the whole test instead of using proper assertions on the response — this often accidentally swallows real failures instead of correctly verifying the expected failure occurred.
- Throwing generic `Error` objects everywhere instead of typed custom errors, losing the ability to distinguish error types (a network error vs. a validation error vs. an unexpected app crash) programmatically.
- Not narrowing `unknown` caught errors with `instanceof` before accessing custom properties — attempting to access `.status` directly on an `unknown`-typed catch variable is a TypeScript compile error, a common early stumbling block.
- Silently catching and ignoring errors ("swallowing" them) without re-throwing or asserting on them — this can make a test pass even though something genuinely broke mid-execution.

## Interview Notes

- Be ready to write a custom error class and explain what context it should carry beyond a plain message string.
- Understand how to properly test a negative/error path — verifying the expected failure occurred with the right assertions, rather than a fragile try/catch-around-the-test pattern.
- Be able to explain why caught errors are typed `unknown` in strict TypeScript and how to safely narrow them — a specific, common practical question for candidates with real strict-mode experience.

## References

- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [MDN — Error](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error)