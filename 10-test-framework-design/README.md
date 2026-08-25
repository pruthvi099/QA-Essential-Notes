# 10 — Test Framework Design

Framework architecture — layering, page object hierarchies, abstraction, logging, configuration, dependency injection, and long-term maintainability. This is where the individual patterns from earlier folders (POM, fixtures, config) come together into full framework design. Read [02-automation-python-playwright](../02-automation-python-playwright/) and [03-typescript-playwright](../03-typescript-playwright/) first.

## Notes

1. [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md) — Layered architecture and the dependency-direction principle
2. [Designing a Scalable Page Object Hierarchy](./designing-a-scalable-page-object-hierarchy.md) — BasePage, component objects, avoiding "God objects"
3. [Abstraction Layers & Driver Wrappers](./abstraction-layers-and-driver-wrappers.md) — Decoupling tests from the underlying automation tool
4. [Logging Strategy for Test Frameworks](./logging-strategy-for-test-frameworks.md) — Structured log levels and what's worth logging
5. [Custom Test Utilities & Helpers](./custom-test-utilities-and-helpers.md) — Building a reusable utility library without over-engineering
6. [Framework Configuration Architecture](./framework-configuration-architecture.md) — Layered config with a clear precedence order
7. [Dependency Injection in Test Frameworks](./dependency-injection-in-test-frameworks.md) — How fixtures already implement DI, and when to go further
8. [Test Framework Maintainability & Technical Debt](./test-framework-maintainability-and-technical-debt.md) — Recognizing and managing framework rot
9. [Choosing a Test Framework From Scratch](./choosing-a-test-framework-from-scratch.md) — Evaluation criteria for a new project

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — the building blocks this folder architects around
- [07 — CI/CD](../07-ci-cd/) — where framework configuration and reporting connect to pipelines