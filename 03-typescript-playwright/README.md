# 03 — TypeScript Playwright (TS-Specific)

TypeScript-only topics that don't have a meaningful Python parallel — language fundamentals for automation, type-safe framework patterns, and TS/Node-specific Playwright tooling. For core Playwright concepts covered in both languages, see [02-automation-python-playwright](../02-automation-python-playwright/).

## Notes

1. [TypeScript Fundamentals for Test Automation](./typescript-fundamentals-for-test-automation.md) — Types, interfaces, async/await, generics for test code
2. [Interfaces & Type-Safe Test Data](./interfaces-and-type-safe-test-data.md) — Modeling test data and building typed data factories
3. [Generics & Reusable Test Utilities](./generics-and-reusable-test-utilities.md) — Writing reusable, type-safe helpers and API client wrappers
4. [Enums & Configuration Management](./enums-and-configuration-management.md) — Typed environment configuration, enums vs. union types
5. [Error Handling & Custom Exceptions](./error-handling-and-custom-exceptions.md) — Custom error classes and correctly testing negative paths
6. [Async Patterns & Promise Handling](./async-patterns-and-promise-handling.md) — Await sequencing, `Promise.all()`, and the click-and-wait-for-event pattern
7. [Modules & Project Structure](./modules-and-project-structure.md) — Organizing a scalable TypeScript automation framework
8. [Custom Matchers & Test Reporting Types](./error-classes-and-custom-matchers.md) — Extending `expect()` with domain-specific assertions
9. [tsconfig & Linting for Test Frameworks](./tsconfig-and-linting-for-test-frameworks.md) — Strict mode and Playwright-specific lint rules
10. [Mocking & Network Interception](./mocking-and-network-interception.md) — Simulating API responses, errors, and slow networks with `page.route()`
11. [Config-Driven Test Parameterization](./typescript-config-driven-test-parameterization.md) — Data-driven tests and multi-browser `projects`
12. [Playwright Component Testing](./playwright-component-testing.md) — Testing React/Vue/Svelte components in isolation
13. [Visual Regression Testing](./visual-regression-testing.md) — Screenshot comparison with `toHaveScreenshot()`
14. [Custom Reporters & HTML Report](./custom-reporters-and-html-report.md) — Built-in and custom Playwright Test reporters
15. [Test Annotations & Tagging](./test-annotations-and-tagging.md) — `skip`/`fixme`/`fail`/`slow` and tag-based test filtering
16. [Debugging & VS Code Integration](./debugging-and-vscode-integration.md) — Inspector, UI Mode, and live local debugging

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — core Playwright concepts covered in both languages
- [10 — Test Framework Design](../10-test-framework-design/) — where these patterns combine into a full framework architecture