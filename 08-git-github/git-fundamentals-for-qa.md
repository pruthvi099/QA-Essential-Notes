# Git Fundamentals for QA

## What It Is

Git is the version control system nearly every test automation codebase lives in — commits, branches, merges, and the local/remote relationship between a developer's machine and a shared repository. This note covers the core mental model an SDET needs daily, not exhaustive Git internals.

## Why It Matters

- Test code is real code, and it lives in the same repository (or a closely related one) as the application — Git fluency isn't optional infrastructure knowledge, it's a daily-use skill for committing, branching, and collaborating on automation the same way developers do.
- Misunderstanding Git's core model (what a commit actually is, how branches relate to each other) leads to real, disruptive mistakes — lost work, tangled history, accidental force-pushes over a teammate's changes.
- This is foundational for every other note in this folder — branching strategy, PR workflow, code review, and conflict resolution all assume the core mental model covered here.

## How It Works

**Core concepts:**
- **Commit** — a snapshot of the repository's state at a point in time, with a message describing the change. Commits form a chain (history), each pointing to its parent.
- **Branch** — a movable pointer to a specific commit; creating a branch doesn't copy files, it just creates a new pointer, which is why branching in Git is fast and cheap.
- **Working directory / staging area / repository** — the three-stage model: your actual files (working directory), what's queued for the next commit (`git add`, staging area), and the committed history (repository).
- **Remote** — a version of the repository hosted elsewhere (GitHub, GitLab); `git push`/`git pull` sync your local repository with it.
- **Merge vs. Rebase** — two different ways to bring one branch's changes into another. Merge creates a new commit joining both histories (preserving exact history); rebase replays one branch's commits on top of another (producing a linear history, but rewriting commit identities).

## Example

A typical daily workflow for adding a new automated test, showing the core commands in context:

```bash
# Start from an up-to-date main branch
git checkout main
git pull origin main

# Create a feature branch for the new test work
git checkout -b add-checkout-discount-tests

# ... write test code ...

# Stage and commit the change
git add tests/checkout/discount-codes.spec.ts
git commit -m "Add discount code validation tests for checkout"

# Push the branch to the remote, creating it there
git push -u origin add-checkout-discount-tests

# Later, after review feedback, add another commit
git add tests/checkout/discount-codes.spec.ts
git commit -m "Address review feedback: add boundary test for exact threshold"
git push
```

Illustrating merge vs. rebase with the same scenario — bringing `main`'s latest changes into a feature branch before opening a PR:

```bash
# MERGE approach: creates a new "merge commit" joining both histories
git checkout add-checkout-discount-tests
git merge main
# History shows a merge commit; both branches' original commits are preserved as-is

# REBASE approach: replays your branch's commits on top of the latest main
git checkout add-checkout-discount-tests
git rebase main
# History is linear, as if your commits were written after main's latest —
# but your commits now have NEW identities (different commit hashes)
```

## Production Considerations

- Most teams adopt a consistent convention (merge vs. rebase) for keeping feature branches up to date with `main`, rather than mixing both freely — inconsistency here creates confusing, hard-to-follow history.
- Never rebase (or force-push) a branch that others have already pulled/based work on — rebasing rewrites commit history, and force-pushing a rewritten branch that a teammate has built on top of causes serious, disruptive conflicts for them.
- Commit test code changes with the same discipline as application code — small, focused commits with clear messages (see [Writing Good Commits & PR Descriptions](./writing-good-commits-and-pr-descriptions.md)) make it far easier to later identify which change introduced a regression, including test-suite regressions.

## Common Pitfalls

- Confusing `git merge` and `git rebase` outcomes — not understanding that rebase rewrites commit hashes is a common source of confusion and, if force-pushed on a shared branch, real disruption to collaborators.
- Committing directly to `main` instead of working in a feature branch — this bypasses code review entirely and is a common mistake for those newer to team-based Git workflows.
- Not pulling the latest `main` before starting new work, leading to avoidable merge conflicts later that a simple `git pull` at the start would have prevented.
- Writing vague, unhelpful commit messages ("fix stuff," "wip") that provide no useful context when someone (including future-you) needs to understand why a change was made.

## Interview Notes

- Be ready to explain the merge vs. rebase distinction clearly, including why rebase rewrites commit history and the collaboration risk that creates if misused on shared branches.
- Understand the three-stage model (working directory, staging area, repository) and what `git add` vs. `git commit` each actually do.
- Be able to describe a typical daily Git workflow for adding a new test — branch, commit, push, PR — showing practical, hands-on fluency rather than just command memorization.

## References

- [Git — Official Documentation](https://git-scm.com/doc)
- [Atlassian — Git Tutorials](https://www.atlassian.com/git/tutorials)