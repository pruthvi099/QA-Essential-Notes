# Code Review for Test Code

## What It Is

Code review for test code applies the same rigor as reviewing application code, but with a distinct set of concerns: test correctness, coverage completeness, maintainability of the automation itself, and avoiding anti-patterns specific to test code (flaky waits, brittle locators, over-mocking). This extends [Test Case Review & Peer Review](../01-manual-testing/test-case-review-and-peer-review.md)'s manual-test-case review discipline into reviewing actual automated test code in a PR.

## Why It Matters

- Test code that passes review poorly (a flaky wait pattern, a brittle locator) doesn't just risk one bad test — it sets a precedent other engineers copy, since new tests are frequently written by pattern-matching existing ones in the codebase.
- Reviewers who only check "does the test pass" miss the deeper questions: does it actually verify the right thing, is it testing at the right level (see [Levels of Testing](../00-start-here/levels-of-testing.md)), will it be maintainable as the app evolves.
- This is a practical, everyday SDET responsibility — reviewing teammates' test PRs is often a bigger part of the job than writing every test personally, especially on larger teams, making review skill directly valuable.

## How It Works

**What to specifically look for in a test code review, beyond "does it pass":**

1. **Correctness of what's being verified** — does the assertion actually check the right thing, or does it just confirm *something* happened without verifying it's *correct* (see [Request & Response Validation](../04-api-testing/request-response-validation.md) for the same principle at the API layer)?
2. **Locator/selector quality** — role/accessibility-based locators preferred over brittle CSS/XPath (see [Locators](../02-automation-python-playwright/locators.md)).
3. **Proper waiting/assertion patterns** — web-first assertions, not manual sleeps (see [Auto-Waiting](../02-automation-python-playwright/auto-waiting.md), [Assertions](../02-automation-python-playwright/assertions.md)).
4. **Test isolation** — does the test create its own data/state, or does it depend on state left by another test (see [Browser Contexts & Test Isolation](../02-automation-python-playwright/browser-contexts-and-test-isolation.md))?
5. **Appropriate test level** — is this tested at the right level (unit/API/E2E), or is a slow UI test verifying something a fast API test could check instead (see [Levels of Testing](../00-start-here/levels-of-testing.md))?
6. **Coverage completeness** — are boundary/negative cases included, or only the happy path (see [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md))?
7. **Naming and readability** — does the test name clearly describe the scenario, so a failure is understandable from the test report alone without opening the code?

## Example

A realistic review comment thread on a test PR, illustrating the kind of feedback that goes beyond "looks good":

```typescript
// Original code in the PR under review
test('checkout works', async ({ page }) => {
  await page.goto('/checkout');
  await page.locator('#card-num').fill('4242424242424242');
  await page.click('#pay-btn');
  await page.waitForTimeout(2000);
  const success = await page.locator('.success-msg').isVisible();
  expect(success).toBe(true);
});
```

```text
Review comments:

1. Naming: "checkout works" is too vague — if this fails, the test
   report doesn't tell you WHAT specifically broke. Suggest renaming
   to something like "checkout completes successfully with a valid card".

2. Locators: #card-num and #pay-btn are CSS ID selectors tied to
   implementation. Prefer role/label-based locators
   (getByLabel('Card Number'), getByRole('button', {name: 'Pay'}))
   for stability — see Locators note.

3. Flaky wait pattern: waitForTimeout(2000) is a hardcoded sleep,
   which is either too short (flaky under slower conditions) or
   wastes time when the app responds faster. Use a web-first
   assertion instead: await expect(page.getByText('Payment
   successful')).toBeVisible() — see Auto-Waiting / Assertions notes.

4. Coverage gap: only the happy path is covered here. Is there a
   companion test for a DECLINED card? That's arguably higher-value
   coverage than the happy path alone (see Negative Testing for APIs
   for the same principle applied to checkout's payment flow).
```

The corrected version after addressing feedback:
```typescript
test('checkout completes successfully with a valid card', async ({ page }) => {
  await page.goto('/checkout');
  await page.getByLabel('Card Number').fill('4242424242424242');
  await page.getByRole('button', { name: 'Pay' }).click();
  await expect(page.getByText('Payment successful')).toBeVisible();
});

test('checkout shows a clear error with a declined card', async ({ page }) => {
  await page.goto('/checkout');
  await page.getByLabel('Card Number').fill('4000000000000002');   // test-mode declined card
  await page.getByRole('button', { name: 'Pay' }).click();
  await expect(page.getByText('Your card was declined')).toBeVisible();
});
```

## Production Considerations

- Establish a lightweight, shared checklist (similar in spirit to the categories above) for test code review specifically — this keeps review quality consistent across different reviewers and prevents review depth from depending entirely on individual reviewer experience.
- Review feedback that identifies a broader anti-pattern (e.g., hardcoded sleeps used throughout the file, not just the one line under review) is worth flagging as a pattern, not just fixing the single instance — anti-patterns in test code tend to get copied forward into future tests otherwise.
- Balance thoroughness with review velocity — not every test needs the exhaustive treatment shown above; scale review depth to the risk/complexity of what's being tested (see [Risk-Based Testing](../00-start-here/risk-based-testing.md)), same as testing effort itself.

## Common Pitfalls

- Reviewing test PRs only for "does it pass in CI" without examining the actual test logic, locator quality, or coverage completeness — this misses most of the value a genuine review provides.
- Approving a test PR that introduces a known anti-pattern (hardcoded sleep, brittle locator) because "it works," without considering that other engineers will likely copy the pattern into future tests.
- Giving vague review feedback ("this could be better") instead of specific, actionable suggestions tied to a concrete principle (as in the example above) — this is the test-code equivalent of the vague defect reports covered in [Defect Reporting Best Practices](../01-manual-testing/defect-reporting-best-practices.md).
- Not flagging missing negative/boundary test coverage during review, since it's easy to focus only on whether the submitted test itself is well-written, rather than whether the submission as a whole is sufficiently complete.

## Interview Notes

- Be ready to review a piece of sample test code live and identify issues — a very common practical interview exercise, often combining several concepts from earlier notes (locators, waits, coverage).
- Understand why test code review needs different scrutiny than application code review — specifically, watching for anti-patterns that get copied forward into future tests.
- Be able to describe how you'd give specific, actionable review feedback rather than vague comments — this is a communication skill interviewers value alongside technical review ability.

## References

- [Google — Code Review Developer Guide](https://google.github.io/eng-practices/review/)
- [Playwright — Best Practices](https://playwright.dev/docs/best-practices)