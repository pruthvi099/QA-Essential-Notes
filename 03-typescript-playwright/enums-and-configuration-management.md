# Enums & Configuration Management

## What It Is

Enums (short for "enumerations") define a fixed, named set of possible values — used in test automation for things like environment names, roles, or status values where a plain string invites typos. This note also covers how enums pair with environment-driven configuration, extending [Hooks & Configuration](../02-automation-python-playwright/hooks-and-config.md) with TypeScript-specific type safety for config values.

## Why It Matters

- A string literal like `"staging"` typed by hand can be mistyped (`"stagng"`) and TypeScript won't catch it — an enum (or a union type) makes invalid values a compile-time error instead of a runtime surprise, often discovered only when a test silently points at the wrong environment.
- Configuration mistakes (wrong base URL, wrong environment) are a common, costly category of test framework bugs — running a suite against the wrong environment can range from wasted time to, worse, accidentally running destructive tests against production.
- This ties together type safety (this folder) with the practical config patterns from `playwright.config.ts` (see [Hooks & Configuration](../02-automation-python-playwright/hooks-and-config.md)), showing how they combine in a real framework.

## How It Works

**Enums:**
```typescript
enum Environment {
  Local = 'local',
  Staging = 'staging',
  Production = 'production',
}
```

**Note:** TypeScript also supports plain union types (`'local' | 'staging' | 'production'`) for the same purpose. Many modern TypeScript style guides prefer union types over enums for simple cases — enums generate actual runtime JavaScript code, while union types are purely a compile-time construct with no runtime footprint. Both are shown here since both are common in real codebases.

**Environment-driven config** — reading `process.env` and validating it against a known set of allowed values:
```typescript
function getEnvironment(): Environment {
  const env = process.env.TEST_ENV;
  if (env !== 'local' && env !== 'staging' && env !== 'production') {
    throw new Error(`Invalid TEST_ENV: ${env}`);
  }
  return env as Environment;
}
```

## Example

A typed configuration module driving `playwright.config.ts`, ensuring an invalid environment fails fast and loudly instead of silently defaulting to the wrong URL:

```typescript
// config/environments.ts
export type Environment = 'local' | 'staging' | 'production';

interface EnvConfig {
  baseURL: string;
  apiURL: string;
}

const environments: Record<Environment, EnvConfig> = {
  local: {
    baseURL: 'http://localhost:3000',
    apiURL: 'http://localhost:4000/api',
  },
  staging: {
    baseURL: 'https://staging.example.com',
    apiURL: 'https://api-staging.example.com',
  },
  production: {
    baseURL: 'https://example.com',
    apiURL: 'https://api.example.com',
  },
};

export function getConfig(): EnvConfig {
  const env = process.env.TEST_ENV as Environment;

  if (!environments[env]) {
    throw new Error(
      `Invalid or missing TEST_ENV: "${process.env.TEST_ENV}". Expected one of: ${Object.keys(environments).join(', ')}`
    );
  }

  return environments[env];
}
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';
import { getConfig } from './config/environments';

const config = getConfig();   // fails immediately and clearly if TEST_ENV is invalid

export default defineConfig({
  use: {
    baseURL: config.baseURL,
  },
});
```

```bash
# Fails fast with a clear error — instead of silently running
# against the wrong environment or an undefined base URL
TEST_ENV=stagng npx playwright test
# Error: Invalid or missing TEST_ENV: "stagng". Expected one of: local, staging, production
```

## Production Considerations

- Validate environment configuration at startup (fail fast), not lazily when the first test happens to use it — a misconfigured environment should stop the entire run immediately with a clear error, not produce confusing failures partway through.
- Never let a config default silently to `production` if an environment variable is missing or invalid — an explicit, loud failure is far safer than an accidental run against a live environment.
- Keep environment-specific secrets (API keys, credentials) out of the `environments.ts` file itself — reference them via `process.env` at the point of use, and manage actual secret values through CI's secret storage, not committed config files.

## Common Pitfalls

- Using plain string literals for environment/role/status values throughout a codebase instead of a typed enum or union — this reintroduces the exact typo risk that TypeScript's type system exists to prevent.
- Letting an invalid or missing environment variable silently fall through to a default (especially a production default) instead of throwing a clear, immediate error.
- Defining environment config values in multiple places (some in `playwright.config.ts` directly, some in a separate file) instead of one single, typed source of truth — this causes drift where different parts of the framework use inconsistent values.
- Choosing enums for every fixed-value case without considering whether a simpler union type would serve just as well with less generated runtime code — a design choice worth making deliberately rather than by default.

## Interview Notes

- Be ready to explain the difference between a TypeScript `enum` and a union type, including the runtime-code distinction — a specific, fairly common question that shows real TypeScript depth beyond surface syntax.
- Understand why environment configuration should fail fast and loudly on an invalid value, rather than defaulting silently — a good example of defensive framework design, not just a TypeScript feature.
- Be able to describe how you'd structure typed, environment-driven configuration for a framework that needs to run against local, staging, and production.

## References

- [TypeScript Handbook — Enums](https://www.typescriptlang.org/docs/handbook/enums.html)
- [Playwright — Environment Variables](https://playwright.dev/docs/test-parameterize)