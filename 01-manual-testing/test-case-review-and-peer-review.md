# Test Case Review & Peer Review

## What It Is

Test case review is the practice of having someone other than the author examine test cases before execution — checking for gaps, ambiguity, incorrect assumptions, and missing edge cases. It's the testing-artifact equivalent of code review: a second set of eyes catches issues the original author is too close to the work to see.

## Why It Matters

- A single author's test cases reflect that person's mental model of the system — their blind spots become coverage gaps unless someone else reviews the cases before execution begins, not after a defect escapes to production.
- Review is far cheaper than the alternative: finding a missing test case *after* a production incident means the gap is discovered at the worst possible time and cost.
- As an SDET, you'll review both manual test cases and automated test code (PRs) — the review skills transfer directly, and interviewers often ask how you'd approach reviewing someone else's test coverage, not just writing your own.

## How It Works

**What a test case review checks for:**
- **Completeness** — are all equivalence partitions and boundaries covered (see [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md))? Are negative/error paths included, not just the happy path?
- **Clarity** — could a different tester execute this test case and get an unambiguous result (see [Writing Effective Test Cases](./writing-effective-test-cases.md))?
- **Correctness** — is the expected result actually correct per the requirement, not just what the author assumed?
- **Traceability** — does every requirement have at least one corresponding test case, and does every test case map back to a requirement?
- **Redundancy** — are there test cases that overlap so heavily they're not adding coverage value?

**A lightweight review process:**
1. Author completes test case design for a feature/story.
2. Reviewer (another tester, or a developer for technical accuracy) reads through independently.
3. Reviewer leaves comments on gaps, ambiguity, or incorrect assumptions — ideally in the same tool the tests live in (test management tool, or PR comments if test cases live in the repo).
4. Author addresses feedback; for significant gaps, a second pass may be warranted.

## Example

A test case review comment thread, showing the kind of gaps a reviewer should be catching:

```text
Test Case Under Review: TC_DISCOUNT_005
Verify discount code SAVE10 applies 10% off orders over ₹500

Steps:
  1. Add item worth ₹600 to cart
  2. Enter code "SAVE10" at checkout
  3. Click Apply
Expected: 10% discount applied, new total is ₹540

---
Reviewer comment (Coverage gap):
  This only covers the valid, above-threshold case. Missing:
    - Order exactly at ₹500 (boundary — is it inclusive or exclusive?)
    - Order below ₹500 (should be rejected)
    - Expired or invalid code
    - Code applied twice (clicking Apply repeatedly)
  Per Boundary Value Analysis, the boundary case (exactly ₹500) is the
  highest-risk gap here and should be a separate test case.

Reviewer comment (Ambiguity):
  Step 3 says "Click Apply" — but doesn't specify what happens if the
  cart total changes AFTER the code is applied (e.g., user removes an
  item, dropping below ₹500). Is the discount expected to be revoked?
  This should either be a separate test case or the requirement needs
  clarification from product.

Author response:
  Added TC_DISCOUNT_006 (boundary, exactly ₹500), TC_DISCOUNT_007
  (below threshold, rejected), TC_DISCOUNT_008 (expired code).
  Flagged the "discount revoked on total drop" question to product —
  behavior wasn't specified in the original requirement.
```

This mirrors code review practice closely enough that the same discipline applies to reviewing automated test code:

```python
# A reviewer would flag this automated test the same way — it only
# covers the happy path, missing the boundary and negative cases
# identified in the manual review above
def test_discount_applies_above_threshold(page):
    page.goto("https://shop.example.com/checkout")
    add_item_to_cart(page, price=600)
    apply_discount_code(page, "SAVE10")
    assert get_total(page) == 540

# Missing, per the same review feedback:
# - test_discount_at_exact_threshold
# - test_discount_rejected_below_threshold
# - test_discount_rejected_expired_code
```

## Production Considerations

- Test case review should happen *before* execution begins, ideally during sprint planning or as soon as test cases are drafted — reviewing after execution has already started wastes the time spent executing incomplete or wrong test cases.
- For teams that keep test cases as code (e.g., BDD feature files, or test case definitions in the repo), review naturally folds into the existing PR review process — no separate tool or workflow needed.
- Review findings that reveal a genuinely ambiguous or unspecified requirement (like the discount-revocation question above) should go back to product/business, not be silently resolved by the tester's own guess — this is a common source of tests that pass but don't verify the "right" behavior.

## Common Pitfalls

- Treating review as a rubber stamp (approving without genuinely checking coverage/clarity) — this defeats the purpose and gives false confidence in coverage.
- Reviewing only for grammar/formatting instead of substantive coverage gaps, missing boundaries, and incorrect assumptions.
- Skipping review entirely under time pressure — this is exactly when review is most valuable, since rushed test case design is where gaps are most likely to appear.
- Not distinguishing between a test case bug (something the reviewer should fix) and a requirement ambiguity (something that needs to go back to product) — conflating the two either delays the review unnecessarily or lets a real requirement gap slip through unresolved.

## Interview Notes

- Be ready to review a given test case live and identify missing coverage — a common practical interview exercise, often paired with the [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md) exercise.
- Be able to explain how test case review parallels code review, and why that similarity matters for how SDET teams should integrate it into their existing workflow rather than treating it as a separate process.
- Understand how to distinguish "the test case has a gap" from "the requirement itself is ambiguous" and describe how you'd route each differently.

## References

- [ISTQB Foundation Level Syllabus — Test Case Development and Review](https://www.istqb.org/certifications/certified-tester-foundation-level)