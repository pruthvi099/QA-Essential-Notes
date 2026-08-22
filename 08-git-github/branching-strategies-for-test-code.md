# Branching Strategies for Test Code

## What It Is

A branching strategy defines how a team organizes concurrent work in Git — when to branch, how long branches live, and how they merge back together. This note covers the two most common models (feature branching, trunk-based development) specifically through the lens of test code: automated tests have a distinct lifecycle concern that application code doesn't always share — a new test is often written *for* a feature that doesn't exist yet, or needs to track a feature branch closely.

## Why It Matters

- Test code branching decisions directly affect CI signal quality — if test branches diverge too far from `main` for too long, the "is this suite currently passing" signal on `main` becomes stale or misleading.
- The choice between feature branching and trunk-based development affects how [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md) and [Smoke vs. Regression in CI](../07-ci-cd/smoke-vs-regression-in-ci.md) actually get exercised — a team's branching model and its CI pipeline design are deeply linked, not independent decisions.
- Interviewers ask about branching strategy specifically to understand whether a candidate has worked in a real, collaborative codebase with deliberate workflow conventions, versus only ever working solo or on throwaway scripts.

## How It Works

**Feature branching** — each unit of work (a feature, a bug fix, a new test suite) gets its own branch, created from `main`, merged back via a reviewed pull request once complete.
- Pros: isolates work cleanly, PR review gate before merging, clear history of what changed and why.
- Cons: branches can live long and drift from `main`, increasing merge conflict risk the longer they're open.

**Trunk-based development** — developers commit small, frequent changes directly to (or via very short-lived branches quickly merged into) a single main branch (`trunk`/`main`), often behind feature flags for incomplete work.
- Pros: minimizes drift and merge conflicts, keeps CI signal on `main` very fresh and reliable.
- Cons: requires more discipline (small commits, feature flags for incomplete work) and often stronger automated test coverage to safely support frequent direct changes to `main`.

**Where test code specifically fits:**
- Tests for an in-progress feature typically live on the same feature branch as the feature itself, merging together.
- Tests for already-existing functionality (new coverage, refactored framework code) can often be added via their own short-lived branch, independent of any specific feature work.
- Long-lived feature branches are a common source of "the tests pass on my branch but fail after merging to main" surprises — the longer a branch lives, the more `main` has moved on without it.

## Example

A feature-branch workflow where test code tracks the feature it covers:

```bash
# Feature and its tests live on the SAME branch, merging together
git checkout -b add-gift-card-checkout

# ... developer implements the feature ...
# ... SDET adds tests for it in the same branch/PR ...

git add src/checkout/gift-card.ts tests/checkout/gift-card.spec.ts
git commit -m "Add gift card redemption at checkout, with test coverage"
git push -u origin add-gift-card-checkout
# Opens ONE PR containing both the feature and its tests together
```

A trunk-based approach for the same feature, using a feature flag to allow safe, frequent small commits directly toward `main`:

```typescript
// The feature is built incrementally behind a flag, and tests are
// added incrementally alongside it — both merge to main frequently
// in small pieces, rather than accumulating on a long-lived branch

test('gift card field is hidden when feature flag is off', async ({ page }) => {
  await page.goto('/checkout?ff_gift_cards=false');
  await expect(page.getByLabel('Gift Card Code')).not.toBeVisible();
});

test('gift card field is visible when feature flag is on @quarantined', async ({ page }) => {
  // Tagged quarantined initially since the feature is still incomplete —
  // see Flaky Test Quarantine in CI — un-quarantined once feature-complete
  await page.goto('/checkout?ff_gift_cards=true');
  await expect(page.getByLabel('Gift Card Code')).toBeVisible();
});
```

A common problem this note is meant to help avoid — a long-lived branch surprise:
```text
Scenario: A feature branch with new checkout tests has been open for
3 weeks. During that time, `main` had a checkout refactor merged.

Result: The branch's tests pass locally (against the OLD checkout
structure) but fail immediately after merging to main, since they
were never run against the refactored code until merge.

Mitigation: Regularly rebase/merge `main` into long-lived branches
(see Git Fundamentals for QA) so CI on the branch reflects an
up-to-date baseline, not a 3-week-old one.
```

## Production Considerations

- Regardless of which model a team uses, keep feature branches (and their accompanying tests) as short-lived as practically possible — the longer a branch lives, the higher the risk of exactly the "passes on branch, fails after merge" scenario shown above.
- For trunk-based development, incomplete features need feature flags specifically so tests for unfinished functionality don't block the main pipeline — this is why the quarantine tagging pattern from [Flaky Test Quarantine in CI](../07-ci-cd/flaky-test-quarantine-in-ci.md) pairs naturally with trunk-based workflows.
- Whichever model is used, it should be a documented, team-wide convention — inconsistent branching practices (some engineers using long feature branches, others committing directly to `main`) create unpredictable CI behavior and confusing history.

## Common Pitfalls

- Letting a feature branch (and its tests) live for weeks without merging `main`'s latest changes into it, leading to a painful, surprising merge/rebase and tests that don't actually reflect current `main` behavior until very late.
- Writing tests for an in-progress feature on a completely separate branch/PR from the feature itself, creating unnecessary coordination overhead and a period where the feature exists without any test coverage tracking it.
- Adopting trunk-based development without adequate feature-flag discipline, resulting in incomplete/broken functionality reaching `main` directly with no isolation mechanism.
- Not having any documented team convention at all, leading to inconsistent branching practices that make the CI pipeline's behavior unpredictable depending on who's committing.

## Interview Notes

- Be ready to compare feature branching and trunk-based development, including the trade-offs each makes around merge conflict risk, CI signal freshness, and required team discipline.
- Understand why long-lived feature branches are a genuine testing risk (stale baseline, surprising post-merge failures), not just a Git hygiene preference.
- Be able to describe how test code for an incomplete feature should be handled under trunk-based development — feature flags plus quarantine tagging is the expected, practical answer.

## References

- [Atlassian — Trunk-Based Development](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development)
- [Martin Fowler — Feature Branch](https://martinfowler.com/bliki/FeatureBranch.html)