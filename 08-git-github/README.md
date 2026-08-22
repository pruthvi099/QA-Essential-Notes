# 08 — Git & GitHub

Version control workflows for test code — branching, review, conflict resolution, and repo hygiene specific to automation projects. Read [00-start-here](../00-start-here/) first for general testing fundamentals; this folder assumes basic Git familiarity, not automation experience.

## Notes

1. [Git Fundamentals for QA](./git-fundamentals-for-qa.md) — Commits, branches, merge vs. rebase, the daily workflow
2. [Branching Strategies for Test Code](./branching-strategies-for-test-code.md) — Feature branching vs. trunk-based development
3. [Writing Good Commits & PR Descriptions](./writing-good-commits-and-pr-descriptions.md) — Commit hygiene and PR context for test changes
4. [Code Review for Test Code](./code-review-for-test-code.md) — What to look for beyond "does it pass"
5. [Resolving Merge Conflicts in Test Files](./resolving-merge-conflicts-in-test-files.md) — Handling conflicts in shared fixtures and data files
6. [Git Bisect for Finding Regressions](./git-bisect-for-finding-regressions.md) — Binary-searching history to find when a test broke
7. [Managing Test Code in Monorepos](./managing-test-code-in-monorepos.md) — Organization and selective test execution at scale
8. [.gitignore & Repo Hygiene for Test Projects](./gitignore-and-repo-hygiene-for-test-projects.md) — What to exclude and why
9. [Git Hooks for Test Quality](./git-hooks-for-test-quality.md) — Catching issues locally, before CI

## Related

- [07 — CI/CD](../07-ci-cd/) — where these workflows connect to pipeline triggers and quality gates
- [10 — Test Framework Design](../10-test-framework-design/) — broader framework organization this folder's practices support