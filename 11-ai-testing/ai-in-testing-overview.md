# AI in Testing: Overview

## What It Is

AI-assisted testing means using large language models and related tools to support QA/SDET work — generating test case drafts, writing automation code snippets, analyzing failures, generating test data — as an engineering capability that augments a tester's judgment, not a replacement for it. This note sets the frame for the rest of the folder: for every AI-assisted technique covered here, the question is always "where does this genuinely help, where does it fail, and what human validation does it still require."

## Why It Matters

- AI in testing is frequently discussed in hype-driven terms ("AI will replace QA") or dismissed entirely — neither framing is accurate or useful; the practical reality is narrower and more specific, and understanding that specificity is what separates genuine capability from marketing.
- Misapplying AI assistance (trusting generated test code without review, treating AI-suggested coverage as complete) introduces real risk — confidently wrong test assertions or missed edge cases are more dangerous than no automation at all, since they create false confidence.
- Interviewers increasingly ask about AI tool usage in testing workflows — being able to describe *specific, bounded* use cases with their limitations (rather than vague enthusiasm or dismissal) signals genuine, current hands-on experience.

## How It Works

**Where AI genuinely helps in testing work today:**
- Drafting test case scenarios from a requirement, as a starting point for human review (see [AI-Assisted Test Case Generation](./ai-assisted-test-case-generation.md)).
- Writing boilerplate automation code (a first-pass page object, a locator suggestion) that a human then verifies and refines.
- Summarizing/clustering large volumes of failure logs to spot patterns faster (see [AI-Assisted Defect Analysis](./ai-assisted-defect-analysis.md)).
- Generating synthetic test data variations quickly (see [AI-Assisted Test Data Generation](./ai-assisted-test-data-generation.md)).

**Where it reliably fails or needs heavy scrutiny:**
- Judging *what's actually worth testing* for a given business — AI lacks the product/business context a human tester builds over time.
- Verifying its own output is correct — an AI can generate a test that runs and passes while asserting the wrong thing entirely, since it has no way to independently confirm business-logic correctness.
- Replacing exploratory testing's human curiosity and judgment (see [Exploratory Testing](../01-manual-testing/exploratory-testing.md)) — AI-generated coverage tends to reflect common patterns, not the creative, adversarial thinking that finds genuinely novel bugs.

**The standing rule for this whole folder:** every AI-assisted output is a draft requiring human review, not a finished deliverable — the specific verification method differs per technique (covered in each note), but the principle doesn't change.

## Example

A concrete illustration of the "helps vs. fails" distinction, using the same task attempted two ways:

```text
Task: Generate test cases for a password reset feature.

WHERE AI HELPS: Given the requirement text, an LLM can quickly draft
a reasonable first pass — valid email, invalid email, expired token,
already-used token, etc. This is a genuinely useful starting point
that saves initial drafting time.

WHERE HUMAN REVIEW IS STILL REQUIRED: The AI draft won't know that
THIS application's password reset tokens are valid for exactly 15
minutes (a detail only in an internal doc or the code itself, not
inferable from a generic requirement description), or that there's
a known edge case where a user requests two reset emails in quick
succession — this is exactly the kind of product-specific, tribal-
knowledge detail that requires a human who knows the system.

RESULT: The AI draft is genuinely useful as a starting checklist,
but treating it as complete, submitted-as-is coverage would miss
real, known risk areas specific to this application.
```

## Production Considerations

- Establish a team norm that AI-generated test code and test cases go through the same review process as human-written ones (see [Code Review for Test Code](../08-git-github/code-review-for-test-code.md)) — AI output shouldn't get a lighter review bar just because it was fast to produce.
- Be deliberate about what context an AI tool has access to — a tool with no visibility into your application's actual codebase/requirements will produce more generic, less useful output than one with proper context, but broader context access also raises its own privacy/security considerations (see [AI Testing Limitations & Risks](./ai-testing-limitations-and-risks.md)).
- Track where AI assistance is actually saving time versus where it's creating rework (correcting confidently-wrong output) — this is genuinely tool- and task-specific, and worth evaluating with real data rather than assumption.

## Common Pitfalls

- Treating AI-generated test coverage as complete without human review, missing product-specific edge cases and tribal knowledge no generic model would know.
- Swinging to the opposite extreme — dismissing AI assistance entirely as unreliable, missing genuine time savings on well-suited tasks (boilerplate generation, first-draft test cases).
- Not distinguishing between different AI-assisted tasks' actual reliability — treating "AI can help draft a test case" and "AI can verify a test's correctness" as equally trustworthy, when they're not.
- Failing to disclose or consider what data/code is being sent to an external AI tool, a genuine concern covered in depth in [AI Testing Limitations & Risks](./ai-testing-limitations-and-risks.md).

## Interview Notes

- Be ready to describe specific, bounded ways you've used (or would use) AI in testing work, including what you still verify manually — vague enthusiasm ("AI is great for testing") is a weak answer; specificity is what interviewers are listening for.
- Understand and be able to articulate why AI can't reliably judge test *sufficiency* or business-logic correctness on its own — this is the core limitation underlying every note in this folder.
- Be able to describe a scenario where blindly trusting AI-generated test output would have caused a real problem — shows practical, considered judgment rather than either uncritical enthusiasm or reflexive skepticism.

## References

- [ISTQB — AI Testing (Foundation Extension)](https://www.istqb.org/certifications/artificial-intelligence-tester)