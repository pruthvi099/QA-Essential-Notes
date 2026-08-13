# Interfaces & Type-Safe Test Data

## What It Is

Beyond basic typing (see [TypeScript Fundamentals for Test Automation](./typescript-fundamentals-for-test-automation.md)), interfaces are the primary tool for modeling test data and API contracts in a TypeScript automation framework. This note goes deeper into interface composition (extending, nested types, optional fields) and the pattern of building **type-safe test data factories** — functions that generate valid, typed test data on demand instead of hardcoding it inline.

## Why It Matters

- As a framework grows, test data gets reused and slightly varied across dozens of tests — without a consistent, typed shape, it's easy to accidentally pass malformed data that a test doesn't catch until a confusing runtime failure occurs.
- Interfaces double as living documentation of your application's data contracts (what a User, Order, or API response actually looks like) — when the backend changes a field, updating one interface surfaces every place in the test suite that needs updating, via compile errors, instead of failing silently at runtime.
- Type-safe test data factories are a common, practical pattern SDETs are expected to build — interviewers sometimes ask candidates to design one live, since it demonstrates framework-thinking beyond individual test scripts.

## How It Works

**Extending interfaces** — build specific types from a shared base without duplication:
```typescript
interface BaseUser {
  id: number;
  email: string;
}

interface AdminUser extends BaseUser {
  permissions: string[];
}
```

**Optional fields** (`?`) — mark fields that may not always be present:
```typescript
interface Product {
  id: number;
  name: string;
  discountCode?: string;   // optional — not every product has one
}
```

**Nested types** — model real-world nested data structures accurately:
```typescript
interface Order {
  id: number;
  customer: BaseUser;
  items: Product[];
}
```

**Test data factories** — functions that produce valid, typed objects with sensible defaults, allowing overrides for the specific fields a test cares about:
```typescript
function buildUser(overrides: Partial<BaseUser> = {}): BaseUser {
  return {
    id: Math.floor(Math.random() * 100000),
    email: `test_${Date.now()}@example.com`,
    ...overrides,
  };
}
```

`Partial<T>` is a TypeScript utility type that makes every field of `T` optional — exactly what's needed for an "overrides" parameter, since a caller shouldn't be forced to specify every field just to override one.

## Example

A realistic test data factory setup for an e-commerce test suite:

```typescript
// types.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  category: 'electronics' | 'clothing' | 'home';
}

export interface Customer {
  id: number;
  email: string;
  tier: 'standard' | 'premium';
}

export interface Order {
  id: number;
  customer: Customer;
  items: Product[];
  total: number;
  discountCode?: string;
}
```

```typescript
// factories.ts
import { Product, Customer, Order } from './types';

let idCounter = 1;

export function buildProduct(overrides: Partial<Product> = {}): Product {
  return {
    id: idCounter++,
    name: 'Test Product',
    price: 999,
    category: 'electronics',
    ...overrides,
  };
}

export function buildCustomer(overrides: Partial<Customer> = {}): Customer {
  return {
    id: idCounter++,
    email: `test_${Date.now()}@example.com`,
    tier: 'standard',
    ...overrides,
  };
}

export function buildOrder(overrides: Partial<Order> = {}): Order {
  const items = overrides.items ?? [buildProduct()];
  const total = items.reduce((sum, p) => sum + p.price, 0);

  return {
    id: idCounter++,
    customer: buildCustomer(),
    items,
    total,
    ...overrides,
  };
}
```

```typescript
// tests/discount.spec.ts
import { test, expect } from '@playwright/test';
import { buildOrder, buildProduct, buildCustomer } from '../factories';

test('premium customer gets discount applied', async ({ page }) => {
  // Only override what THIS test actually cares about —
  // everything else gets sensible, type-checked defaults
  const order = buildOrder({
    customer: buildCustomer({ tier: 'premium' }),
    items: [buildProduct({ price: 1500 })],
    discountCode: 'PREMIUM10',
  });

  // TypeScript would catch a mistake like tier: 'gold' at compile
  // time, since 'gold' isn't part of the Customer.tier union type
  expect(order.customer.tier).toBe('premium');
});
```

## Production Considerations

- Keep factories composable — a `buildOrder()` factory calling `buildCustomer()` and `buildProduct()` internally, each independently overridable, scales far better than one giant factory function with dozens of parameters.
- Centralize interfaces that mirror real API contracts, and treat changes to them as a signal to check for breaking changes across the test suite — this is where TypeScript's compile-time checking pays for itself directly in maintenance time saved.
- Use union types (`'standard' | 'premium'`) instead of plain `string` for fields with a fixed set of valid values — this prevents typos (`'premuim'`) from silently compiling and only failing at runtime.

## Common Pitfalls

- Hardcoding test data inline in every test instead of using factories — this duplicates setup logic and makes bulk changes (e.g., adding a new required field) tedious and error-prone across dozens of files.
- Typing a field as plain `string` when it actually only has a few valid values (status, role, tier) — a union type catches invalid values at compile time that a plain string type would silently allow.
- Making factory overrides required instead of using `Partial<T>` — this forces every caller to specify every field, defeating the convenience factories are meant to provide.
- Not keeping factory-generated IDs unique across a test run, causing subtle test data collisions, especially problematic under parallel execution (see [Parallel Execution](../02-automation-python-playwright/parallel-execution.md)).

## Interview Notes

- Be ready to design a type-safe test data factory live, given a simple interface — a common, practical exercise that tests both TypeScript fluency and framework-design thinking.
- Understand `Partial<T>` and why it's the right tool for factory override parameters specifically.
- Be able to explain how interfaces mirroring API contracts help catch breaking backend changes early, via compile errors rather than runtime test failures.

## References

- [TypeScript Handbook — Interfaces](https://www.typescriptlang.org/docs/handbook/2/objects.html)
- [TypeScript Handbook — Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)