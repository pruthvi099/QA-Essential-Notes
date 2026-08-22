# Git Hooks for Test Quality

## What It Is

Git hooks are scripts that run automatically at specific points in the Git workflow — `pre-commit` (before a commit is created), `pre-push` (before pushing to a remote) — letting a team catch issues locally, before code even reaches CI. This note covers using hooks specifically to enforce lint checks and fast test runs early, shifting feedback even further left than [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md)'s fastest CI stage.

## Why It Matters

- A CI failure costs a round-trip: push, wait for the pipeline, see the failure, fix, push again — a pre-commit/pre-push hook catches the same class of issue in seconds, locally, before that round-trip ever starts.
- This is the practical, tooling-level expression of "shift-left" testing (see [SDLC Models](../00-start-here/sdlc-models.md)) — catching a problem at the earliest possible point in the workflow, literally before the code leaves the developer's machine.
- Hooks are commonly set up and maintained by SDETs specifically, since deciding what's fast/reliable enough to run pre-commit (versus what belongs only in CI) is a testing-strategy judgment call, not just a scripting task.

## How It Works

**Common hook types relevant to test quality:**
- **`pre-commit`** — runs before a commit is finalized; typically used for very fast checks (lint, formatting, type-checking) since it runs on every single commit and must stay near-instant to avoid frustrating developers.
- **`pre-push`** — runs before pushing to a remote; can afford slightly more time than pre-commit (runs less frequently — once per push, not once per commit), making it suitable for a fast subset of tests (e.g., smoke tests) in addition to lint/type checks.

**Tooling:** raw Git hooks live in `.git/hooks/` but aren't version-controlled by default (that directory isn't tracked) — most teams use a wrapper tool (**Husky** for Node.js/TypeScript projects, **pre-commit** framework for Python) that manages hooks as versioned, shareable configuration committed to the repository.

## Example

**TypeScript — Husky configuration running lint on pre-commit and smoke tests on pre-push:**
```bash
npm install --save-dev husky
npx husky init
```

```bash
# .husky/pre-commit
npx eslint . --max-warnings=0
npx tsc --noEmit
```

```bash
# .husky/pre-push
npx playwright test --grep @smoke
```

**Python — using the `pre-commit` framework, configured declaratively:**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8

  - repo: local
    hooks:
      - id: smoke-tests
        name: Run smoke tests
        entry: pytest -m smoke
        language: system
        pass_filenames: false
        stages: [pre-push]   # only on push, NOT on every commit — too slow for that
```

```bash
pip install pre-commit --break-system-packages
pre-commit install --hook-type pre-commit --hook-type pre-push
```

A deliberate boundary decision worth documenting — what stays out of pre-commit specifically:
```text
Decision: Full E2E smoke suite runs on pre-PUSH, not pre-COMMIT.

Reasoning: Developers commit frequently (small, incremental commits)
but push less often. A ~2-minute smoke suite on every commit would be
disruptive; on every push, it's an acceptable trade-off for catching
critical-path breakage before it even reaches CI.

Full regression suite is NOT in any hook — it belongs only in CI
(see Pull Request Test Triggers), since it's too slow for any
local, blocking workflow.
```

## Production Considerations

- Keep `pre-commit` hooks extremely fast (lint, type-check, formatting only) — anything that meaningfully slows down every single commit will be perceived as friction and encourages developers to bypass hooks (`git commit --no-verify`) rather than fix the underlying issue.
- `pre-push` can reasonably afford a fast, narrow test subset (smoke tests specifically) since it runs less frequently than pre-commit — but should still stay well under a minute or two to avoid becoming a workflow annoyance.
- Always provide an escape hatch (`--no-verify`) for genuine emergencies, but treat frequent use of it by the team as a signal that hooks are miscalibrated (too slow, too strict) rather than assuming developers are just being careless.

## Common Pitfalls

- Putting anything slow (a full test suite, a full regression run) into a `pre-commit` hook, making every single commit painfully slow and training developers to bypass hooks entirely with `--no-verify`.
- Not using a shared, version-controlled hook management tool (Husky, `pre-commit` framework), leaving hooks as an individual developer's local, unshared setup that new team members never get and existing checks silently drift out of sync.
- Duplicating slow, CI-appropriate checks (full regression) into hooks "for extra safety," when CI already covers this and the hook only adds local friction without proportional benefit.
- Not documenting the reasoning behind what's included/excluded from hooks (as shown in the example) — a future engineer might either bypass hooks not understanding their purpose, or add something inappropriately slow without realizing the deliberate speed constraint.

## Interview Notes

- Be ready to explain the difference between `pre-commit` and `pre-push` hooks, including why they typically hold different levels of check intensity (pre-commit must be near-instant; pre-push can afford slightly more).
- Understand why hooks need a shared, version-controlled management tool (Husky, `pre-commit` framework) rather than relying on individual developers' local, untracked hook setups.
- Be able to describe how hooks connect to the broader "shift-left" testing philosophy — catching issues even before they reach CI, extending the fast-feedback principle from [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md) one step further left.

## References

- [Husky — Documentation](https://typicode.github.io/husky/)
- [pre-commit — Documentation](https://pre-commit.com/)
- [Git — Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)