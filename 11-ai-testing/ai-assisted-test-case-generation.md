# AI-Assisted Test Case Generation

## What It Is

This note covers using an LLM to draft test case scenarios from a requirement or feature description — extending [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md)'s systematic techniques (Equivalence Partitioning, Boundary Value Analysis) with AI as a first-pass drafting tool, followed by deliberate human validation against those same established techniques.

## Why It Matters

- Drafting an initial set of test scenarios from a requirement is time-consuming, and an LLM can generate a reasonable first pass quickly — genuinely saving time on the *drafting* step, while leaving the *validation* step (which is where real testing judgment lives) to the human.
- The risk isn't that AI-generated test cases are useless — it's that they can look complete and well-organized while silently missing product-specific edge cases, exactly the gap discussed in [AI in Testing: Overview](./ai-in-testing-overview.md).
- Being able to systematically validate AI-generated test cases against known techniques (EP/BVA, requirement traceability) — rather than either blindly accepting or reflexively distrusting them — is a practical, current skill.

## How It Works

**Effective prompting for test case generation:** the more specific and complete the input (the actual requirement text, any known business rules, existing similar test cases as examples), the better the output — vague prompts ("write test cases for login") produce generic, low-value output missing anything specific to your actual application.

**The validation checklist for AI-generated test cases** (applying [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md) and [Test Case Review & Peer Review](../01-manual-testing/test-case-review-and-peer-review.md) deliberately):
1. Does it cover all equivalence partitions and boundaries for the actual, specific business rule (not just generic ones)?
2. Does it include negative/error paths, not just happy-path scenarios?
3. Does it reflect any product-specific rules or exceptions the AI couldn't have known (check against actual requirements/code, not just plausibility)?
4. Are the "expected results" actually correct, or just plausible-sounding? (AI can generate a confident, wrong expected result just as easily as a correct one.)

## Example

A prompt and the resulting AI-generated draft, followed by the human validation pass that catches what it missed:

```text
Prompt given to an LLM:
"Generate test case scenarios for this requirement: 'Users can apply
one discount code per order. A discount code is valid if it exists,
is not expired, and the order total meets its minimum threshold.'"
```

```text
AI-generated draft (a reasonable starting point):
1. Valid code, order meets minimum threshold → discount applied
2. Valid code, order below minimum threshold → discount rejected
3. Expired code → discount rejected
4. Nonexistent code → discount rejected
5. Empty code field → validation error
```

```text
Human validation pass — checking against Test Case Design Techniques
and actual application-specific knowledge:

✅ EP/BVA coverage: mostly good, but MISSING the boundary case
   (order total EXACTLY at the minimum threshold) — added as #6

🐞 Product-specific gap AI couldn't have known: this application
   allows a discount code to be entered, then REMOVED and a different
   one applied — the requirement text doesn't mention this, but it's
   a real user flow per an existing Jira ticket. Added as #7.

🐞 "One discount code per order" implies a test for attempting to
   apply a SECOND code while one is already applied — not obviously
   covered by the draft's happy/negative framing. Added as #8.

Final validated set: 8 test cases (5 from AI draft + boundary case +
2 product-specific cases the AI had no way to know about)
```

## Production Considerations

- Provide the LLM with as much real context as reasonably possible (actual requirement text, not a summary; known edge cases from past incidents) — output quality scales directly with input specificity, and generic prompts produce generic, lower-value drafts.
- Always run AI-generated test cases through the same review process as human-written ones (see [Test Case Review & Peer Review](../01-manual-testing/test-case-review-and-peer-review.md)) — treating AI drafts as exempt from review because they're already "written" is a common, risky shortcut.
- Track over time whether AI-assisted drafting is actually saving meaningful time once the validation pass is included — for some teams/tasks the net time saved is substantial; for others, the validation overhead approaches the time it would have taken to draft manually, and this is worth evaluating honestly rather than assuming.

## Common Pitfalls

- Submitting AI-generated test cases without validation, missing product-specific edge cases and tribal knowledge no generic model could have known.
- Using vague, low-context prompts and getting correspondingly generic output, then being surprised the coverage doesn't reflect the actual application's real behavior.
- Assuming AI-stated "expected results" are correct without independently verifying them against actual requirements or application behavior — a confidently wrong expected result is a real risk, not a hypothetical one.
- Not applying the same equivalence partitioning/boundary value discipline to validate AI output that would be applied to reviewing any human-written test case draft.

## Interview Notes

- Be ready to describe a concrete workflow for using AI to draft test cases, including the specific validation steps you'd apply afterward — the validation half of the answer is what interviewers are really listening for.
- Understand why AI-generated test cases can look complete while missing product-specific context, and be able to give a concrete example of the kind of gap that creates.
- Be able to connect this technique back to established test design methodology (EP/BVA) — showing AI assistance as an extension of rigorous testing practice, not a replacement for it.

## References

- [ISTQB — AI Testing (Foundation Extension)](https://www.istqb.org/certifications/artificial-intelligence-tester)