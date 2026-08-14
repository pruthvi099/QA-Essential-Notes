# tsconfig & Linting for Test Frameworks

## What It Is

`tsconfig.json` controls how the TypeScript compiler checks your code — which errors it catches, how strict it is, and what JavaScript it targets. ESLint (with `@typescript-eslint`) adds a further layer of static analysis beyond type-checking — catching stylistic issues, common bug patterns, and Playwright-specific anti-patterns before code is ever run. Together, they're the configuration layer that determines how much a TypeScript test framework actually protects you from mistakes.

## Why It Matters

- A loosely configured `tsconfig.json` (with `strict: false`, implicit `any` allowed) gives a false sense of type safety — the code "is TypeScript," but many of the compile-time guarantees this whole folder has covered (catching typos, missing awaits, invalid unions) simply don't fire without strict mode enabled.
- Lint rules catch real, common Playwright mistakes automatically — a missing `await` (see [Async Patterns & Promise Handling](./async-patterns-and-promise-handling.md)) is exactly the kind of bug a properly configured lint rule flags before code review even happens.
- Framework configuration is often overlooked in favor of writing tests, but it's a one-time investment that pays off on every single test written afterward — interviewers assessing SDET/framework maturity sometimes ask directly what lint rules or tsconfig settings a candidate would enable and why.

## How It Works

**Key `tsconfig.json` settings for a test framework:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["tests/**/*.ts", "pages/**/*.ts", "fixtures/**/*.ts"]
}
```

`strict: true` enables a bundle of checks, most notably:
- `strictNullChecks` — `null`/`undefined` must be explicitly handled, not silently assumed away
- `noImplicitAny` — every value must have an inferred or explicit type, not a silent fallback to `any`

**Key ESLint rules for Playwright specifically** (via `eslint-plugin-playwright`):
- `playwright/missing-playwright-await` — flags a Playwright method call missing `await`
- `playwright/no-conditional-in-test` — flags `if`/branching logic inside a test (a common anti-pattern that makes tests non-deterministic)
- `@typescript-eslint/no-floating-promises` — flags any promise that isn't awaited or otherwise handled, catching the same class of bug more generally

## Example

A realistic, minimal config pairing for a Playwright TypeScript framework:

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint', 'playwright'],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:playwright/recommended',
  ],
  rules: {
    '@typescript-eslint/no-floating-promises': 'error',
    'playwright/missing-playwright-await': 'error',
    'playwright/no-conditional-in-test': 'warn',
  },
};
```

What strict mode + lint catches that loose config wouldn't:

```typescript
// With strict: false and no lint rules, ALL of these compile
// and run without any warning:
function getUserEmail(user) {          // implicit `any` — no type checking at all
  return user.emial;                    // typo silently allowed, returns undefined
}

page.goto('/checkout');                 // missing await — fires without waiting
if (someCondition) {                    // conditional logic in a test — non-deterministic
  await page.click('#special-button');
}
```

```typescript
// With strict: true + the lint rules above, each of these is
// caught immediately — as a compile error or a lint error —
// before the test is ever run:
function getUserEmail(user: User): string {
  return user.email;                    // typo now impossible — `emial` doesn't exist on User
}

await page.goto('/checkout');           // await enforced by lint rule

// Conditional logic flagged by playwright/no-conditional-in-test —
// prompting a rethink (e.g., splitting into two separate tests)
```

## Production Considerations

- Enable `strict: true` from the very start of a new framework — retrofitting strict mode onto an existing large, loosely-typed codebase later surfaces a wave of errors all at once, which is far more disruptive than starting strict.
- Run both `tsc --noEmit` (type-check only) and `eslint` as required checks in CI, not just locally — catching these issues only in a developer's editor (if they even have the right extensions installed) means they can still reach the main branch otherwise.
- `eslint-plugin-playwright`'s rules encode real, common Playwright mistakes the Playwright team itself has identified — enabling its recommended rule set is a low-effort, high-value starting point rather than hand-picking rules from scratch.

## Common Pitfalls

- Leaving `strict: false` (or omitting it, since it defaults to false) "to move faster initially" — this consistently costs more time later once real bugs that strict mode would have caught reach a running test suite instead.
- Installing `@typescript-eslint` and `eslint-plugin-playwright` but never actually wiring them into CI — lint rules that only run optionally/locally get skipped under time pressure and stop providing real protection.
- Treating `noImplicitAny` violations as annoying friction to silence with explicit `any` types, instead of actually typing the value correctly — this defeats the purpose of enabling strict mode in the first place.
- Not including `playwright/no-conditional-in-test` (or an equivalent review discipline) — conditional logic inside tests is a common, subtle source of non-deterministic, hard-to-debug test behavior.

## Interview Notes

- Be ready to explain what `strict: true` actually enables (name at least `strictNullChecks` and `noImplicitAny` specifically) and why it matters for a test framework in particular.
- Understand what `eslint-plugin-playwright` catches that generic TypeScript/ESLint rules wouldn't — this shows awareness of Playwright-specific tooling, not just general TypeScript best practices.
- Be able to explain why conditional logic inside a test is considered an anti-pattern, and what to do instead (e.g., split into separate, deterministic tests).

## References

- [TypeScript — tsconfig Reference](https://www.typescriptlang.org/tsconfig)
- [eslint-plugin-playwright](https://github.com/playwright-community/eslint-plugin-playwright)
- [typescript-eslint](https://typescript-eslint.io/)