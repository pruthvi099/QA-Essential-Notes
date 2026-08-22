# Managing Test Code in Monorepos

## What It Is

A monorepo is a single repository containing multiple, often independently deployable projects (multiple services, a frontend and backend, several apps) rather than each living in its own separate repository. This note covers the test-code-specific concerns that come with monorepos: organizing test code across projects, and — critically — running only the tests relevant to what actually changed, rather than the entire monorepo's test suite on every change.

## Why It Matters

- In a monorepo, a change to one small service shouldn't require running every other project's entire test suite — without deliberate selective-execution strategy, CI runtime balloons as the monorepo grows, directly undermining the fast-feedback goals covered in [CI/CD Fundamentals for QA](../07-ci-cd/ci-cd-fundamentals-for-qa.md) and [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md).
- Monorepos are increasingly common at companies with many interrelated services — this is a genuinely practical, modern concern many SDETs encounter, not a niche or outdated topic.
- Test code organization in a monorepo needs deliberate structure (shared utilities vs. project-specific tests) to avoid both duplication (the same helper reimplemented per project) and inappropriate coupling (one project's tests accidentally depending on another's internals).

## How It Works

**Common monorepo test organization patterns:**
- **Co-located tests** — each project/package has its own `tests/` directory alongside its source code, keeping tests close to what they verify.
- **Shared test utilities package** — genuinely reusable helpers (a shared API client wrapper, common fixtures) live in their own package, imported by each project's tests, avoiding duplication (see [Modules & Project Structure](../03-typescript-playwright/modules-and-project-structure.md) for the underlying module organization principles, applied here at the monorepo scale).

**Selective test execution (the core monorepo-specific CI concern):**
- **Path-based filtering** — CI determines which projects' tests to run based on which file paths changed in a given commit/PR, using either the CI platform's native path filters or a monorepo tool (Nx, Turborepo, Bazel) that understands the dependency graph.
- **Dependency-graph-aware execution** — more sophisticated tools don't just look at directly changed paths, but also run tests for any project that *depends on* what changed (e.g., a shared library change triggers tests in every project using that library), which naive path-based filtering alone would miss.

## Example

**GitHub Actions — simple path-based filtering, running only a service's tests when its own directory changes:**
```yaml
name: Service Tests

on:
  pull_request:
    paths:
      - 'services/checkout/**'
      - 'shared/test-utils/**'   # also trigger if the shared utility changes

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci --workspace=services/checkout
      - run: npx playwright test --config=services/checkout/playwright.config.ts
```

**Using Nx (a common monorepo tool) to run tests only for affected projects, including dependency-graph awareness a simple path filter would miss:**
```bash
# Runs tests ONLY for projects affected by the changes in this PR,
# where "affected" includes projects that DEPEND ON a changed shared
# library — not just projects with directly changed files
nx affected --target=test --base=main
```

A monorepo test structure showing the co-located + shared pattern:
```text
monorepo/
├── services/
│   ├── checkout/
│   │   ├── src/
│   │   └── tests/                    # co-located with the checkout service
│   ├── inventory/
│   │   ├── src/
│   │   └── tests/                    # co-located with the inventory service
├── shared/
│   ├── test-utils/                   # genuinely reusable across services
│   │   ├── api-client.ts
│   │   └── fixtures.ts
```

## Production Considerations

- Investing in a monorepo-aware tool (Nx, Turborepo, Bazel) pays off once the monorepo has enough projects that naive "run everything on every change" becomes a genuine CI bottleneck — for a small monorepo (a handful of projects), simple path-based filtering is often sufficient without the added tooling complexity.
- Be deliberate about what goes into a shared test utilities package versus staying project-specific — over-sharing creates coupling where a change intended for one project's tests unexpectedly affects others; under-sharing causes duplicated, drifting implementations of the same helper logic.
- Path-based filtering alone (without dependency-graph awareness) can create a dangerous blind spot: a change to a shared library might not trigger tests in the projects that depend on it, if those projects' own paths didn't change — this is exactly the gap dependency-graph-aware tooling closes.

## Common Pitfalls

- Running the entire monorepo's test suite on every single PR regardless of what changed, causing CI times to grow linearly (or worse) with the monorepo's total size rather than with the size of any individual change.
- Using naive path-based filtering without dependency-graph awareness, silently missing test coverage for downstream projects affected by a shared library change — a real, consequential gap, not just an inefficiency.
- Over-centralizing test utilities into an overly broad "shared" package that different projects don't actually use consistently, creating an illusion of reuse while different teams quietly diverge in practice anyway.
- Not having any deliberate test organization convention in a growing monorepo, leading to inconsistent patterns across projects that make it hard for engineers moving between projects to find and understand test code.

## Interview Notes

- Be ready to explain why running an entire monorepo's test suite on every change doesn't scale, and describe at least one selective-execution strategy (path-based filtering, or a dependency-graph-aware tool) to address it.
- Understand the distinction between path-based filtering and dependency-graph-aware execution, and be able to give a concrete example of a bug the former would miss that the latter would catch (a shared library change affecting downstream consumers).
- Be able to describe how you'd organize test code and shared utilities across multiple projects in a monorepo, balancing reuse against unwanted coupling.

## References

- [Nx — Affected Commands](https://nx.dev/ci/features/affected)
- [Google — Why Google Stores Billions of Lines of Code in a Single Repository](https://research.google/pubs/why-google-stores-billions-of-lines-of-code-in-a-single-repository/)