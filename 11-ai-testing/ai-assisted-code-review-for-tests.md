# AI-Assisted Code Review for Tests

## What It Is

This note covers using AI tools as a *first-pass* reviewer of test code — catching obvious anti-patterns (hardcoded waits, brittle locators) automatically before a human reviewer's time is spent on it — explicitly positioned as a supplement to, not a replacement for, the human review discipline covered in [Code Review for Test Code](../08-git-github/code-review-for-test-code.md).

## Why It Matters

- AI-assisted first-pass review can catch mechanical, pattern-matchable issues (a hardcoded sleep, a CSS selector where a role-based locator would be better) quickly and consistently, freeing human reviewer attention for the things AI genuinely can't judge: whether the test verifies the *right* thing, whether coverage is actually complete for the business scenario.
- This is a meaningfully different task from [AI-Assisted Test Automation Code](./ai-assisted-test-automation-code.md) (generating code) — reviewing existing code for known anti-patterns is a narrower, more reliable AI task than generating entirely new, correct code from scratch, and it's worth understanding why that reliability difference exists.
- Teams increasingly integrate AI review tools into their PR workflow — understanding what these tools catch well versus what still requires a human is practical, current knowledge for working within (and setting expectations around) such a workflow.

## How It Works

**What AI-assisted review is genuinely reliable at (pattern-matchable, mechanical issues):**
- Detecting hardcoded sleeps/waits.
- Flagging CSS/XPath selectors where role/accessibility-based locators would be more appropriate.
- Catching missing `await` keywords or other syntactic/structural issues.
- Identifying overly broad locators likely to match multiple elements.

**What AI-assisted review is NOT reliable at (requires human judgment):**
- Whether the test actually verifies the correct business behavior — this requires understanding the requirement, not just the code's structure.
- Whether coverage is complete for the scenario — an AI reviewing one test file in isolation has no way to know what other tests exist or what the full risk profile of the feature actually is.
- Whether a locator choice, while technically fine, will actually be stable given the *real* application's DOM (an AI reviewing code without live access to the running app can't verify this).

**A practical workflow:** AI-assisted review runs automatically (as a CI check, or an IDE/PR integration) catching mechanical issues before a human ever looks at the PR; the human reviewer then focuses their limited time on correctness, coverage, and business-logic judgment — the two review passes are complementary, not redundant.

## Example

An AI-assisted review comment (mechanical, pattern-based) alongside the kind of comment only a human reviewer caught — illustrating the genuine division of labor:

```typescript
// Code under review
test('checkout completes', async ({ page }) => {
  await page.goto('/checkout');
  await page.locator('#card-num').fill('4242424242424242');
  await page.click('#pay-btn');
  await page.waitForTimeout(2000);
  const success = await page.locator('.success-msg').isVisible();
  expect(success).toBe(true);
});
```

```text
AI-assisted review comments (fast, mechanical, pattern-based):
  - Line 4: Consider using page.getByLabel() or getByRole() instead
    of CSS selector '#card-num' for better locator stability.
  - Line 6: waitForTimeout() detected — consider using a web-first
    assertion (expect().toBeVisible()) instead of a hardcoded wait.
  - Line 7: Extracting a boolean from isVisible() then asserting
    separately loses Playwright's built-in retry behavior.

Human reviewer comment (requires actual product/business judgment,
something an AI reviewing this file in isolation cannot assess):
  - This test only covers the SUCCESSFUL payment path. Given that
    payment failure handling was the subject of a recent production
    incident (INCIDENT-204), this PR should also include a declined-
    card test case before merging. [This is a coverage-completeness
    judgment call requiring knowledge the AI reviewer doesn't have
    access to.]
```

This shows the practical split: AI catches the mechanical anti-patterns fast and consistently; the human catches the coverage gap that requires actual context about the application's risk history.

## Production Considerations

- Configure AI-assisted review tools to run automatically and early (as a fast CI check or pre-commit hook, connecting to [Git Hooks for Test Quality](../08-git-github/git-hooks-for-test-quality.md)) — this catches mechanical issues before a human reviewer's time is spent on them, rather than duplicating that effort.
- Set clear team expectations that AI review comments address code *pattern* quality, while human reviewers remain responsible for correctness and coverage — without this explicit framing, teams risk either over-trusting AI review as sufficient, or under-utilizing it by having humans re-check things AI already caught reliably.
- Periodically evaluate whether the AI review tool's suggestions are actually accurate and aligned with the team's specific standards (see [Code Review for Test Code](../08-git-github/code-review-for-test-code.md)) — a misconfigured or poorly-tuned tool generating noisy, low-value comments trains reviewers to ignore it entirely.

## Common Pitfalls

- Treating AI-assisted review as sufficient on its own, skipping human review entirely — this misses exactly the coverage/correctness judgment calls AI reviewing code in isolation cannot make.
- Having human reviewers spend time re-flagging the same mechanical issues (hardcoded waits, brittle locators) an AI tool already reliably catches, wasting review bandwidth that should go toward business-logic and coverage judgment instead.
- Not tuning or validating the AI review tool against the team's actual standards, leading to noisy or inaccurate suggestions that erode trust in the tool over time.
- Assuming AI review "passing" (no comments) means the test code is genuinely good — absence of mechanical anti-pattern flags says nothing about whether the test actually verifies the right thing.

## Interview Notes

- Be ready to explain the specific division of labor between AI-assisted review (mechanical, pattern-based) and human review (correctness, coverage, business judgment) — this distinction is the core, expected insight.
- Understand why AI review is more reliable at catching known anti-patterns than at judging test sufficiency or business-logic correctness, and be able to explain the reasoning (pattern-matching vs. requiring actual product context).
- Be able to describe how you'd integrate an AI review tool into an existing PR workflow without displacing necessary human review — a practical, current, and increasingly relevant question.

## References

- [GitHub — About GitHub Copilot Code Review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review)