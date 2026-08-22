# Resolving Merge Conflicts in Test Files

## What It Is

A merge conflict occurs when Git can't automatically reconcile changes to the same lines/region of a file from two different branches, requiring manual resolution. This note covers conflict resolution specifically in test files and fixtures — which have some distinct patterns (shared fixture files, parametrized test data arrays, page object locator lists) that conflict differently than typical application code.

## Why It Matters

- Test files — especially shared fixtures, page objects, and parametrized data files — are frequently edited by multiple engineers concurrently (everyone adding new test cases touches the same files), making them a common conflict hotspot.
- Resolving a test-file conflict incorrectly (accidentally dropping one side's test case, or merging locators incorrectly) can silently delete test coverage without anyone immediately noticing — a uniquely costly kind of resolution mistake compared to conflicts in application code, where a mistake often causes an obvious build failure.
- This is a practical, everyday skill on any team with more than one person writing automated tests — conflicts in test files are routine, not exceptional, and handling them confidently (rather than defaulting to "just pick one side and hope") matters.

## How It Works

**What a conflict marker looks like:**
```text
<<<<<<< HEAD
(your current branch's version of this section)
=======
(the incoming branch's version of this section)
>>>>>>> incoming-branch-name
```

**Resolution requires deciding, for each conflicting section:** keep your version, keep the incoming version, or (very common for test files specifically) **combine both** — since two engineers adding different new test cases to the same file usually both deserve to be kept, not chosen between.

**Why test files conflict differently than typical application code:** a shared fixture file or a parametrized test-data array often has many engineers appending new entries near the same location (end of a file, end of an array) — Git's line-based diffing frequently flags these as conflicts even though the actual intent (both additions being valid) is to keep both, not choose one.

## Example

A realistic conflict in a shared, parametrized test data file — two engineers each added a new discount code test case near the end of the same array:

```typescript
// test-data/discountCodes.ts
export const discountTestCases: DiscountTestCase[] = [
  { code: 'SAVE10', orderTotal: 1000, expectedTotal: 900, description: 'valid 10% code' },
  { code: 'SAVE20', orderTotal: 1000, expectedTotal: 800, description: 'valid 20% code' },
<<<<<<< HEAD
  { code: 'WELCOME5', orderTotal: 500, expectedTotal: 475, description: 'new user 5% code' },
=======
  { code: 'HOLIDAY15', orderTotal: 1000, expectedTotal: 850, description: 'holiday 15% code' },
>>>>>>> add-holiday-discount-tests
];
```

**Correct resolution** — both additions are valid, independent test cases; keep both:
```typescript
export const discountTestCases: DiscountTestCase[] = [
  { code: 'SAVE10', orderTotal: 1000, expectedTotal: 900, description: 'valid 10% code' },
  { code: 'SAVE20', orderTotal: 1000, expectedTotal: 800, description: 'valid 20% code' },
  { code: 'WELCOME5', orderTotal: 500, expectedTotal: 475, description: 'new user 5% code' },
  { code: 'HOLIDAY15', orderTotal: 1000, expectedTotal: 850, description: 'holiday 15% code' },
];
```

**A genuinely different-intent conflict**, where combining both isn't correct — two engineers modified the SAME existing entry differently, and this needs a real decision, not a blind merge:

```typescript
<<<<<<< HEAD
  { code: 'SAVE10', orderTotal: 1000, expectedTotal: 900, description: 'valid 10% code (min order updated to 1000)' },
=======
  { code: 'SAVE10', orderTotal: 500, expectedTotal: 450, description: 'valid 10% code' },
>>>>>>> update-minimum-order-threshold
```
```text
This requires actually understanding WHY each change was made —
checking commit messages/PR context (see Writing Good Commits & PR
Descriptions) to determine which reflects the CURRENT correct business
rule, rather than guessing or arbitrarily picking one side.
```

Command-line workflow for resolving and completing a conflicted merge:
```bash
git merge main
# ... conflict occurs, edit the file(s) to resolve manually ...
git add test-data/discountCodes.ts
git commit   # completes the merge with the resolved file
```

## Production Considerations

- After resolving any test file conflict, always re-run the affected tests locally before pushing — a syntactically valid but logically incorrect resolution (e.g., accidentally duplicating or dropping a test object) can pass a naive visual check but still be wrong.
- For frequently-conflicting shared files (large fixture/data files many engineers touch), consider whether the file could be restructured (split by feature area, or use a pattern less prone to line-adjacent conflicts) to reduce conflict frequency going forward.
- When a conflict reflects genuinely different, contradictory intent (as in the second example) rather than two independent additions, resolving it requires understanding *why* each side changed what it did — checking the originating commit messages/PRs (see [Writing Good Commits & PR Descriptions](./writing-good-commits-and-pr-descriptions.md)) rather than guessing.

## Common Pitfalls

- Blindly accepting "your" or "their" version wholesale to make a conflict go away quickly, without checking whether the correct resolution is actually to combine both changes — a common cause of silently lost test coverage.
- Not re-running affected tests after resolving a conflict, missing a resolution mistake (a dropped comma, a duplicated object, an accidentally deleted test case) that looks fine on a visual diff but breaks at runtime.
- Resolving a conflict in a shared fixture/page-object file without considering whether the change affects other tests relying on that same fixture — a resolution that looks locally correct can still have broader ripple effects.
- Treating every test-file conflict as a "combine both" situation by default, without checking for the genuinely-different-intent case (as in the second example) where a real decision, not just concatenation, is required.

## Interview Notes

- Be ready to explain how conflict resolution in test data/fixture files often differs from application code — specifically, the "combine both additions" pattern being far more common than in typical code conflicts.
- Understand why silently losing test coverage during conflict resolution is a distinctly risky failure mode, and what practice (re-running tests after resolving) mitigates it.
- Be able to describe how you'd handle a conflict where both sides made genuinely different, contradictory changes to the same test case — requiring investigation, not blind merging.

## References

- [Git — Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [GitHub — Resolving a Merge Conflict Using the Command Line](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line)