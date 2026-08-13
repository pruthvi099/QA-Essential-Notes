# Generics & Reusable Test Utilities

## What It Is

Generics let you write functions, classes, and types that work across multiple types while still preserving full type safety — instead of either duplicating code per type or falling back to `any` and losing type checking entirely. In test automation, generics show up most often in reusable utility functions: API response wrappers, retry helpers, and data-lookup functions that need to work with many different data shapes.

## Why It Matters

- Without generics, a reusable utility either gets duplicated per type (`findProductById`, `findUserById`, `findOrderById` — all nearly identical) or loses type safety by accepting/returning `any` — generics solve this by keeping one implementation, fully typed, for any type you pass in.
- API testing frameworks especially benefit from generics: a single typed HTTP client wrapper can return `Promise<T>` for any endpoint's response shape, rather than writing a separate wrapper function per endpoint.
- Generics are a common marker of TypeScript fluency in interviews — many candidates can write basic typed functions but haven't needed to reach for generics, so using them appropriately (not over-using them) signals real framework-building experience.

## How It Works

**Basic generic function** — `<T>` is a placeholder type, inferred from what's actually passed in:
```typescript
function wrapInArray<T>(item: T): T[] {
  return [item];
}
wrapInArray(5);        // inferred as number[]
wrapInArray("hello");  // inferred as string[]
```

**Generic constraints** — restrict what `T` can be, when the function needs to rely on some property existing:
```typescript
function findById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}
```

**Generic classes** — useful for wrapping data with consistent, reusable structure (e.g., an API response wrapper):
```typescript
class ApiResponse<T> {
  constructor(public status: number, public data: T) {}

  isSuccess(): boolean {
    return this.status >= 200 && this.status < 300;
  }
}
```

## Example

A generic, typed API client wrapper — a realistic reusable utility in a TypeScript automation framework:

```typescript
// api-client.ts
import { APIRequestContext } from '@playwright/test';

async function getJson<T>(request: APIRequestContext, url: string): Promise<T> {
  const response = await request.get(url);
  if (!response.ok()) {
    throw new Error(`GET ${url} failed with status ${response.status()}`);
  }
  return response.json() as Promise<T>;
}
```

```typescript
// types.ts
export interface Product {
  id: number;
  name: string;
  price: number;
}

export interface Order {
  id: number;
  items: Product[];
  total: number;
}
```

```typescript
// tests/api.spec.ts
import { test, expect } from '@playwright/test';
import { getJson } from '../api-client';
import { Product, Order } from '../types';

test('fetches typed product and order data', async ({ request }) => {
  // Same generic function, used for two completely different shapes —
  // TypeScript knows `product` is a Product and `order` is an Order,
  // with full autocomplete and compile-time checking on each
  const product = await getJson<Product>(request, '/api/products/101');
  const order = await getJson<Order>(request, '/api/orders/55');

  expect(product.price).toBeGreaterThan(0);
  expect(order.items.length).toBeGreaterThan(0);
});
```

A generic `findById` helper applied to test data lookups, using a constraint to require an `id` field:

```typescript
function findById<T extends { id: number }>(items: T[], id: number): T {
  const found = items.find(item => item.id === id);
  if (!found) throw new Error(`No item found with id ${id}`);
  return found;
}

const products: Product[] = [
  { id: 1, name: 'Mouse', price: 899 },
  { id: 2, name: 'Keyboard', price: 1499 },
];

const mouse = findById(products, 1);   // typed as Product, no cast needed
```

## Production Considerations

- Reach for generics when you notice the *same logic* being duplicated across multiple near-identical typed functions — that duplication is the practical signal generics are worth introducing, rather than adding them speculatively upfront.
- Generic constraints (`T extends { id: number }`) are what make generic utilities like `findById` actually useful in practice — an unconstrained `T` can't safely access `.id` at all, since TypeScript can't assume every possible type has that property.
- Don't over-generalize simple, one-off helper functions into generics just because it's possible — if a utility is only ever used with one specific type, plain typing is clearer and easier for other engineers to read than an unnecessary generic.

## Common Pitfalls

- Using `any` instead of a generic when writing a reusable utility that needs to work across multiple types — this silently gives up all the type safety a generic would have preserved.
- Forgetting generic constraints when the function body needs to access a specific property (like `.id`) — without `extends { id: number }`, TypeScript correctly refuses to compile, since it can't guarantee an arbitrary `T` has that field.
- Overusing generics for functions that only ever handle one concrete type — this adds unnecessary complexity and makes the code harder to read without any real reuse benefit.
- Casting with `as` to force a type match instead of properly typing a generic function — this bypasses TypeScript's checking entirely and can hide real bugs (e.g., an API response that doesn't actually match the expected shape).

## Interview Notes

- Be ready to write a simple generic function (like `findById`) live, including the constraint syntax — a common practical TypeScript exercise in SDET interviews.
- Understand why a generic, typed API client wrapper is preferable to writing a separate typed wrapper function per endpoint — this is the clearest practical justification for generics in a testing context.
- Be able to explain when generics are the right tool versus overkill — showing restraint here is as important a signal as knowing the syntax.

## References

- [TypeScript Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)