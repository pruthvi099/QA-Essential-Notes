# Git Bisect for Finding Regressions

## What It Is

`git bisect` is a built-in Git tool that uses binary search across commit history to efficiently pinpoint exactly which commit introduced a regression — you mark a known-good commit and a known-bad commit, and Git automatically checks out commits in between, narrowing down the culprit in roughly log₂(n) steps rather than checking every commit sequentially.

## Why It Matters

- "This test started failing sometime in the last 200 commits, but I don't know which one" is a common, frustrating investigation scenario — `git bisect` turns a potentially huge manual search into a small, fast, systematic one (searching 200 commits takes about 8 steps, not 200).
- This is a genuinely practical, hands-on tool an SDET reaches for during real regression investigation — not theoretical Git knowledge, but something used during actual debugging sessions when the "when did this break" question comes up.
- Combined with automated test execution, `git bisect run` can fully automate the search — letting Git itself run a test at each candidate commit and determine good/bad automatically, without manual intervention at each step.

## How It Works

**Manual bisect workflow:**
1. `git bisect start` — begins the session.
2. `git bisect bad` — mark the current (broken) commit as bad.
3. `git bisect good <commit>` — mark a known-good commit (e.g., last week's release tag).
4. Git checks out a commit halfway between good and bad; you test it and mark it `git bisect good` or `git bisect bad`.
5. Repeat — Git keeps narrowing the range until it identifies the exact first bad commit.
6. `git bisect reset` — ends the session, returning to your original branch/commit.

**Automated bisect (`git bisect run`)** — instead of manually testing and marking each commit, provide a script that exits with code 0 (good) or non-zero (bad); Git runs it automatically at each step, fully automating the search.

## Example

**Manual bisect session investigating when a specific test started failing:**
```bash
git bisect start
git bisect bad HEAD                    # current commit is broken
git bisect good v2.4.0                 # last known-good release tag

# Git checks out a commit halfway between — run the failing test manually
npx playwright test tests/checkout/discount.spec.ts

# If it passes at this commit:
git bisect good

# If it still fails at this commit:
git bisect bad

# ... repeat until Git narrows it down to a single commit ...
# Git outputs: "abc1234 is the first bad commit"

git bisect reset   # return to original branch
```

**Fully automated bisect using `git bisect run`, letting Git drive the entire search:**
```bash
git bisect start
git bisect bad HEAD
git bisect good v2.4.0

# Git automatically checks out each candidate commit, runs this
# command, and uses its exit code to decide good/bad — no manual
# intervention needed at each step
git bisect run npx playwright test tests/checkout/discount.spec.ts --reporter=dot

# Output after the automated search completes:
# abc1234 is the first bad commit
# commit abc1234
# Author: ...
# Date: ...
#     Refactor discount calculation to use new pricing service

git bisect reset
```

A realistic investigation this enables — connecting back to earlier CI/CD concepts:
```text
Scenario: The nightly regression suite (see Smoke vs. Regression in CI)
shows "checkout discount test" has been failing for an unknown number
of nights. The last known-passing nightly run was 12 days ago (~40
commits merged to main since then).

Using git bisect run with the specific failing test as the check
script narrows this down to 6 steps instead of potentially checking
all 40 commits — quickly identifying the exact commit (a discount
calculation refactor) that introduced the regression.
```

## Production Considerations

- `git bisect run` requires a genuinely reliable, deterministic test/check script — if the test itself is flaky (see [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md)), the bisect process can be misled by intermittent false-positive/negative results, producing an incorrect "first bad commit."
- Bisect works best with a reasonably clean, buildable commit history at every point in the range — if some intermediate commits don't build/run at all (common with certain rebase-heavy or work-in-progress histories), those need to be marked as `git bisect skip` rather than good/bad.
- Combining `git bisect run` with a fast, narrowly-scoped test command (just the specific failing test, not the full suite) significantly speeds up the search, since the check runs once per bisect step.

## Common Pitfalls

- Attempting to bisect using a flaky test as the pass/fail check, leading Git to an incorrect conclusion due to the test's own inconsistency rather than a genuine code change.
- Manually checking commits one at a time sequentially instead of using bisect's binary search, wasting significant time on an investigation bisect would resolve far faster.
- Not resetting the bisect session (`git bisect reset`) when done, leaving the repository checked out at an arbitrary historical commit instead of back on the working branch.
- Using the full, slow test suite as the bisect check script when only one specific failing test is actually relevant — this multiplies the time cost of each bisect step unnecessarily.

## Interview Notes

- Be ready to explain what `git bisect` does and why it's more efficient than manually checking commits sequentially — the binary search concept and its log₂(n) efficiency is the core, expected explanation.
- Understand `git bisect run` specifically and how it automates the process using a test script's exit code — a common follow-up showing deeper, practical tool fluency beyond the basic manual workflow.
- Be able to describe a realistic scenario (a test regression discovered days after it was introduced) where bisect would be the practical tool of choice — this grounds the answer in real investigative use, not just command memorization.

## References

- [Git — git-bisect Documentation](https://git-scm.com/docs/git-bisect)
- [Git Book — Debugging with Git](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git)