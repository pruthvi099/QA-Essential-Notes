# Writing Good Commits & PR Descriptions

## What It Is

This note covers the practical craft of writing clear commit messages and pull request descriptions specifically for test code changes — what makes them genuinely useful to a reviewer and to future engineers (including future-you) investigating history, rather than a formality to get through quickly.

## Why It Matters

- Commit history is a debugging tool, not just a changelog — `git blame`/`git log` on a test file is frequently how someone investigates *why* a specific assertion or wait was added, and a vague message ("fix test") gives that investigation nothing to work with.
- A well-written PR description for test changes specifically should answer "what gap did this close, and why does it matter" — this is different from application code PRs, which usually explain a feature or fix; test PRs need to justify the *coverage decision* itself.
- This directly extends [Test Case Review & Peer Review](../01-manual-testing/test-case-review-and-peer-review.md) into the Git/GitHub-specific artifacts (commit messages, PR descriptions) that carry that review context — a reviewer can't assess coverage gaps or ambiguity if the PR gives them no context to evaluate against.

## How It Works

**A good commit message structure:**
```text
<short summary line, imperative mood, under ~50 chars>

<optional longer body explaining WHY, not just what — the diff
already shows what changed>
```

**What makes a commit message for test code specifically useful:**
- State *what scenario* is now covered, not just "add test."
- If fixing a flaky/broken test, explain the root cause found, not just "fix flaky test."
- If removing/skipping a test, explain why — this is exactly the kind of decision a future engineer needs justified, not just observed.

**A good PR description for test changes should answer:**
1. What coverage gap does this close, or what defect does it verify?
2. How was it verified (ran locally, ran against staging, etc.)?
3. Any deliberate scope decisions (what's intentionally NOT covered, and why)?

## Example

**Poor commit message (common, unhelpful):**
```text
fix test
```

**Good commit message for the same change:**
```text
Fix flaky discount code test by waiting for cart total update

The test was asserting on the cart total immediately after clicking
"Apply", before the async recalculation completed. Switched to a
web-first assertion (expect().toHaveText()) which retries until the
updated total appears, instead of a one-time check.
```

**A PR description for new test coverage, following the structure above:**
```markdown
## What this covers

Adds negative-path test coverage for the discount code field at
checkout: expired codes, codes below the minimum order threshold,
and codes with trailing whitespace (a bug found during exploratory
testing — see JIRA-4821).

## How verified

Ran locally against staging (`TEST_ENV=staging`), all 4 new tests
passing. Also ran the full checkout regression suite to confirm no
regressions from the shared fixture change.

## Scope notes

Does NOT cover concurrent application of two discount codes — that's
tracked separately in JIRA-4830, since it requires a backend change
that hasn't shipped yet.
```

**A commit message explaining a deliberate skip — the kind of context a bare `test.skip()` in code doesn't fully capture:**
```text
Skip gift card redemption test pending backend flag rollout

The gift card feature is behind a flag not yet enabled in the CI
staging environment (see JIRA-4790 for rollout timeline). Test is
written and ready — re-enable once the flag defaults to true in
staging config.
```

## Production Considerations

- Treat the PR description as documentation for whoever investigates this change months later, not just a formality for the current reviewer — the "why," including deliberately excluded scope, ages far better as a searchable PR description than as tribal knowledge someone has to ask about.
- For test-only PRs, explicitly stating how the change was verified (ran locally, ran against a specific environment) gives reviewers real confidence — "trust me, it works" is a weaker signal than "ran the full suite against staging, all passing."
- Link PR descriptions back to the originating bug/ticket (as shown above) wherever a test change traces back to a specific defect or requirement — this preserves the traceability discussed in [Test Documentation Overview](../00-start-here/test-documentation-overview.md), now expressed through Git artifacts.

## Common Pitfalls

- Writing commit messages that only restate what the diff already shows ("changed X to Y") instead of explaining *why* — the diff makes "what" obvious; the message's real job is "why."
- Leaving PR descriptions empty or minimal ("added tests") for test-code changes specifically, when reviewers most need context on what coverage decision is being made and why.
- Not documenting the reasoning behind a skipped or deliberately narrow-scoped test, leaving future engineers to either duplicate the investigation or, worse, assume the gap was accidental and unknowingly work around a decision that was actually deliberate.
- Bundling unrelated test changes into a single commit/PR, making the history harder to search and review — the same "one test case verifies one thing" discipline from [Writing Effective Test Cases](../01-manual-testing/writing-effective-test-cases.md) applies to commit scoping too.

## Interview Notes

- Be ready to explain what makes a commit message genuinely useful (the "why," not just the "what") and be able to critique a vague example.
- Understand why test-code PR descriptions specifically should justify coverage/scope decisions, not just describe the change — this is a distinguishing detail from general PR-writing advice.
- Be able to describe a scenario where good commit history directly helped you (or would help someone) debug or understand a past decision — shows practical appreciation for this discipline, not just theoretical agreement.

## References

- [Chris Beams — How to Write a Git Commit Message](https://cbea.ms/git-commit/)
- [GitHub — About Pull Requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests)