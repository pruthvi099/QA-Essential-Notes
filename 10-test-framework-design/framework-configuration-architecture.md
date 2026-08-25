# Framework Configuration Architecture

## What It Is

This note covers designing a layered configuration system at full framework scale — combining sensible defaults, environment-specific overrides, and command-line/CI overrides into one coherent precedence order. It extends [Hooks & Configuration](../02-automation-python-playwright/hooks-and-config.md) and [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md) from individual-project config into the architectural pattern a growing framework needs as it supports more environments, more override scenarios, and more contributors.

## Why It Matters

- As a framework grows to support multiple environments (local, staging, multiple CI contexts) and multiple override needs (a developer temporarily testing against a different URL, a CI job needing environment-specific credentials), an ad-hoc, unlayered config approach becomes error-prone and hard to reason about.
- A clear precedence order (defaults → environment file → CLI/environment variable override) means anyone can predict exactly what value will actually be used in a given run, without needing to trace through scattered conditional logic.
- This is a natural, expected extension of the configuration concepts covered individually throughout the repo — interviewers ask about layered config specifically to see if a candidate can design for the *general* problem (any config value, any environment, any override need) rather than just hardcoding a few environment-specific values.

## How It Works

**The standard precedence order, lowest to highest priority:**
1. **Defaults** — sensible, hardcoded fallback values baked into the framework itself.
2. **Environment-specific config** — values that differ per environment (local/staging/production-adjacent), typically defined in per-environment config files.
3. **Environment variables** — values injected at runtime (from CI secrets, or a developer's local `.env`), which should override environment-file defaults when present.
4. **CLI arguments** — the highest-priority override, for one-off, explicit runs (e.g., "just this once, run against a different URL").

**Each layer overrides the one below it only where it explicitly provides a value** — a config system should never require every value to be specified at every layer; it should fall through to the next lower layer when a layer doesn't specify something.

## Example

A layered configuration system implementing this precedence order explicitly:

```typescript
// config/defaults.ts
export const defaults = {
  timeout: 30000,
  retries: 0,
  headless: true,
};
```

```typescript
// config/environments.ts
export const environmentConfig = {
  local: {
    baseURL: 'http://localhost:3000',
    retries: 0,
  },
  staging: {
    baseURL: 'https://staging.example.com',
    retries: 2,
  },
  production: {
    baseURL: 'https://example.com',
    retries: 2,
    headless: true,   // enforced regardless of local dev preference
  },
};
```

```typescript
// config/resolve-config.ts — implements the full precedence order
import { defaults } from './defaults';
import { environmentConfig } from './environments';

interface ResolvedConfig {
  baseURL: string;
  timeout: number;
  retries: number;
  headless: boolean;
}

export function resolveConfig(): ResolvedConfig {
  const env = (process.env.TEST_ENV ?? 'local') as keyof typeof environmentConfig;
  const envConfig = environmentConfig[env];

  if (!envConfig) {
    throw new Error(`Unknown environment: ${env}`);
  }

  // Layer 1 + 2: defaults, overridden by environment-specific config
  let resolved: ResolvedConfig = { ...defaults, ...envConfig };

  // Layer 3: environment variables override the above, WHERE PRESENT
  if (process.env.BASE_URL) resolved.baseURL = process.env.BASE_URL;
  if (process.env.TEST_TIMEOUT) resolved.timeout = parseInt(process.env.TEST_TIMEOUT);
  if (process.env.HEADLESS === 'false') resolved.headless = false;

  // Layer 4: CLI arguments override everything (highest priority)
  const cliArgs = process.argv.slice(2);
  const headedFlag = cliArgs.includes('--headed');
  if (headedFlag) resolved.headless = false;

  return resolved;
}
```

```bash
# Demonstrating the precedence order in practice:

# Uses defaults + local environment config only
TEST_ENV=local npx playwright test

# Environment variable overrides the environment config's baseURL
TEST_ENV=staging BASE_URL=http://localhost:9999 npx playwright test

# CLI flag overrides EVERYTHING, including the env var/config-derived headless setting
TEST_ENV=production npx playwright test --headed
```

## Production Considerations

- Fail loudly and immediately on an invalid/unknown environment value (as shown with the thrown error above) — the same defensive principle from [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md), now applied at the full layered-config level where more values are involved and a silent fallback is even riskier.
- Document the precedence order clearly (in a framework README) so contributors understand exactly which layer wins in a given scenario — an undocumented precedence order leads to confusing "why isn't my override working" debugging sessions.
- Keep the resolved, final configuration loggable/inspectable at framework startup (minus secrets) — being able to see exactly what configuration a given test run actually resolved to is invaluable for diagnosing "this ran against the wrong environment" issues.

## Common Pitfalls

- Not having a clear, consistent precedence order at all — config values scattered across multiple files with ad-hoc, inconsistent override logic makes it genuinely hard to predict what value will actually be used in any given run.
- Silently ignoring an invalid environment variable value instead of failing loudly — this is the same risk pattern flagged in [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md), and it's more dangerous the more layers/values a config system has.
- Requiring every value to be explicitly specified at every layer, rather than allowing sensible fallthrough to defaults — this makes environment-specific config files unnecessarily verbose and repetitive.
- Not logging the final resolved configuration, making it hard to confirm (especially when debugging a misconfigured CI run) what values were actually used versus what was intended.

## Interview Notes

- Be ready to design a layered configuration system with a clear precedence order (defaults → environment → env vars → CLI) for a hypothetical framework — a common, practical framework-architecture interview question.
- Understand why failing loudly on invalid configuration matters even more at this scale than in a single-file config — more layers and values mean more opportunities for a silent misconfiguration to go unnoticed.
- Be able to explain why logging the final resolved configuration at startup is a valuable, low-cost diagnostic practice.

## References

- [The Twelve-Factor App — Config](https://12factor.net/config)
- [Playwright — Environment Variables](https://playwright.dev/docs/test-parameterize)