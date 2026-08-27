# AI-Assisted Test Automation Code

## What It Is

This note covers using AI coding assistants (GitHub Copilot, ChatGPT, Claude) to generate automation code — locators, page object boilerplate, assertion patterns — and the specific, real failure modes this introduces: hallucinated APIs, outdated syntax, and locator suggestions that violate the best practices covered throughout [02-automation-python-playwright](../02-automation-python-playwright/) and [03-typescript-playwright](../03-typescript-playwright/).

## Why It Matters

- AI coding assistants are genuinely useful for boilerplate (a page object's basic structure, a repetitive fixture pattern) — but automation code has specific correctness requirements (a locator that actually matches the real DOM, a wait pattern that's actually appropriate) that an AI can't verify without actually running against the real application.
- Hallucination — an AI confidently generating a plausible-looking but nonexistent method or API — is a well-documented, real failure mode, and automation code is a place where a hallucinated method name fails loudly (a clear error) but a *subtly wrong* locator or wait pattern can fail silently or intermittently, which is worse.
- This connects directly to [Code Review for Test Code](../08-git-github/code-review-for-test-code.md) — AI-generated automation code needs exactly the same anti-pattern scrutiny (brittle locators, hardcoded waits) as human-written code, arguably more, since AI tends to reproduce common patterns from its training data that may predate current best practices.

## How It Works

**What AI coding assistants are genuinely good at for automation code:**
- Generating repetitive boilerplate (a new page object class following an established pattern already present in the codebase).
- Suggesting a plausible locator strategy when given the actual HTML/DOM structure as context.
- Writing test skeletons that a human then fills in with actual business logic and assertions.

**Common, specific failure modes in AI-generated automation code:**
1. **Hallucinated APIs** — suggesting a method that doesn't exist on the actual library version (e.g., a Playwright method from an older/newer version, or one that was never real).
2. **Outdated patterns** — generating Selenium-era patterns (explicit `time.sleep()`, manual wait loops) even when asked for Playwright code, since older patterns are heavily represented in training data.
3. **Locator anti-patterns** — defaulting to brittle CSS/XPath selectors instead of role/accessibility-based locators (see [Locators](../02-automation-python-playwright/locators.md)), since AI has no way to know whether a given element has a stable `data-testid` or accessible role without being shown the actual markup.
4. **Plausible-but-wrong assertions** — generating an assertion that looks reasonable but doesn't actually verify the intended behavior correctly.

## Example

**A locator anti-pattern an AI assistant commonly generates without specific guidance, and the corrected version:**
```typescript
// AI-GENERATED (without being shown actual page structure) — defaults
// to a brittle CSS selector, exactly the anti-pattern flagged in
// Locators
await page.locator('.btn.btn-primary.submit-order-btn').click();
```

```typescript
// CORRECTED after human review, applying the actual project's
// locator standards
await page.getByRole('button', { name: 'Submit Order' }).click();
```

**An outdated wait pattern an AI assistant might generate from older training data, versus the correct modern approach:**
```typescript
// AI-GENERATED — a Selenium-era pattern that predates Playwright's
// auto-waiting, a common artifact of training data skewing older
await page.click('#submit-btn');
await page.waitForTimeout(3000);   // hardcoded sleep — an anti-pattern
                                     // flagged throughout Auto-Waiting
const successMsg = await page.locator('.success').isVisible();
```

```typescript
// CORRECTED — using Playwright's actual auto-waiting and web-first
// assertions, as covered in Auto-Waiting and Assertions
await page.getByRole('button', { name: 'Submit' }).click();
await expect(page.getByText('Order submitted successfully')).toBeVisible();
```

**A hallucinated API — a real, documented risk category, shown as a caution rather than a specific fabricated example:**
```text
A known risk with AI coding assistants: they can generate a method
call that looks plausible but doesn't actually exist on the library
version in use — e.g., inventing a slightly-wrong method name based
on patterns from similar libraries or older/newer versions.

Mitigation: always verify a suggested method against the actual,
current library documentation before trusting it — a method that
"looks right" isn't the same as a method that's real.
```

## Production Considerations

- Provide AI coding assistants with real context (actual DOM structure, the project's existing page object patterns, the specific library version in use) whenever possible — output quality on locator/API correctness improves significantly with concrete context versus a bare "write a test for X" prompt.
- Treat every AI-suggested method call as unverified until confirmed against actual, current documentation or by successfully running it — this is a low-cost habit that catches hallucinated APIs before they cause confusing runtime errors.
- Apply the project's established locator/wait/assertion standards (from [02-automation-python-playwright](../02-automation-python-playwright/)) as an explicit checklist when reviewing AI-generated code, since AI-generated code is *more* likely (not less) to default to older anti-patterns unless specifically guided otherwise.

## Common Pitfalls

- Accepting AI-suggested locators without checking them against actual best practices (role/accessibility-based over CSS/XPath) — AI tends to default to brittle patterns unless explicitly steered otherwise.
- Trusting a suggested method/API call without verifying it exists on the actual library version in use, risking a hallucinated API that fails at runtime in a confusing way.
- Accepting hardcoded sleep/wait patterns from AI-generated code, reintroducing exactly the flakiness anti-pattern that [Auto-Waiting](../02-automation-python-playwright/auto-waiting.md) and [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md) warn against.
- Giving AI-generated code a lighter code review pass than human-written code "since it was fast to produce" — this is exactly backwards, given AI's specific, documented failure modes.

## Interview Notes

- Be ready to describe specific failure modes of AI-generated automation code (hallucinated APIs, outdated wait patterns, brittle locators) rather than a vague "sometimes it's wrong" — specificity signals real, hands-on experience with these tools.
- Understand why AI-generated code needs the same (or arguably more) scrutiny in code review as human-written code, not less.
- Be able to describe how providing better context (actual DOM structure, existing codebase patterns) improves AI-generated automation code quality — a practical, actionable insight beyond just "AI code needs review."

## References

- [Playwright — Best Practices (Node.js)](https://playwright.dev/docs/best-practices)
- [GitHub — About GitHub Copilot](https://docs.github.com/en/copilot/about-github-copilot)