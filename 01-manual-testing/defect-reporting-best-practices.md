# Defect Reporting Best Practices

## What It Is

Defect reporting is the practice of documenting a found bug clearly enough that a developer can reproduce, understand, and fix it without needing to go back and forth with the reporter. A defect report is a communication artifact between tester and developer — its quality directly determines how fast a bug gets fixed.

This builds on [Bug Life Cycle](../00-start-here/bug-life-cycle.md), which covers the states a defect moves through; this note focuses specifically on how to *write* the report well at the "New" stage.

## Why It Matters

- A vague defect report ("checkout is broken") causes back-and-forth clarification that can take longer than fixing the actual bug — poor reporting is one of the biggest hidden costs in a testing process.
- Good defect reports are also what makes automated test failures actionable — when a CI test fails, the failure output (screenshot, trace, logs) needs to function as a defect report by itself, since no human manually reproduced it first.
- Interviewers assess this skill directly by asking you to write or critique a bug report — it signals whether you think about the *reader* of your report, not just the bug itself.

## How It Works

A strong defect report includes:

| Field | Purpose |
|---|---|
| Title | Specific, searchable summary — not generic |
| Environment | Browser/OS/app version/environment (staging, prod, etc.) |
| Preconditions | State required before reproducing |
| Steps to Reproduce | Numbered, minimal steps — the shortest path that reliably reproduces it |
| Expected Result | What should have happened |
| Actual Result | What actually happened |
| Severity | Impact on the system (crash, data loss, cosmetic, etc.) |
| Priority | Urgency to fix (business-driven, independent of severity) |
| Evidence | Screenshot, video, logs, network trace |
| Reproducibility | How consistently it occurs (e.g., 3/3 times, intermittent) |

**Key writing principles:**
- **Title should be specific**: "Checkout fails with 500 error when cart has ₹0 item" beats "Checkout broken."
- **Steps should be minimal**: strip out anything not required to reproduce — extra steps hide the real trigger and slow debugging.
- **One bug per report**: bundling multiple issues into one report makes tracking, prioritizing, and closing individually impossible.
- **State the actual vs. expected explicitly**: don't make the reader infer what "should" have happened.

## Example

**Poorly written:**
```text
Title: Checkout broken
The checkout page doesn't work when I add items and try to buy stuff.
```

**Well written:**
```text
Title: Checkout returns 500 error when cart contains a ₹0 promotional item

Environment: Staging, Chrome 126.0, macOS 14.5
Severity: High     Priority: P1
Reproducibility: 3/3 (consistent)

Preconditions:
  - User logged in with a standard test account
  - Cart is empty

Steps to Reproduce:
  1. Add promotional item "Free Sample Kit" (₹0) to cart
  2. Add any paid item (e.g., "Wireless Mouse", ₹899) to cart
  3. Go to /checkout
  4. Click "Place Order"

Expected Result:
  Order is placed successfully; confirmation page is shown

Actual Result:
  API returns 500 Internal Server Error; user sees a generic
  "Something went wrong" message; order is NOT created

Evidence:
  - Screenshot attached (error-checkout-500.png)
  - API response body attached (500 response, NullReferenceException
    in OrderTotalCalculator.cs line 42 — see server log excerpt)

Additional Notes:
  - Does NOT reproduce if the ₹0 item is removed from cart
  - Does NOT reproduce with a non-zero-priced promotional item
```

When this comes from an automated test rather than manual exploration, the same information should be captured automatically as CI artifacts attached to the failure:

```python
# Playwright: capture evidence automatically on failure, so the CI
# failure report has everything a manual defect report would have
import pytest

@pytest.fixture(autouse=True)
def capture_evidence_on_failure(request, page):
    yield
    if request.node.rep_call.failed:
        page.screenshot(path=f"failures/{request.node.name}.png")
        with open(f"failures/{request.node.name}_console.log", "w") as f:
            f.write("\n".join(str(m) for m in page.context._impl_obj))
```

## Production Considerations

- Defect reports should link back to the test case or automated test that found them (traceability) — this closes the loop between test design and defect tracking, and helps identify which areas of the suite are actually finding real bugs.
- For automated failures, distinguish "test failed" from "confirmed defect" before filing — flaky/environment-caused failures filed as bugs waste developer time and erode trust in the report queue; triage first.
- Attach machine-readable evidence (API response bodies, stack traces, logs) whenever possible, not just screenshots — screenshots show *that* something's wrong, logs/traces show *why*, and developers need the latter to actually fix it fast.

## Common Pitfalls

- Writing steps that include unnecessary actions, obscuring the actual minimal trigger for the bug.
- Omitting environment details, leading to "works on my machine" disputes that waste cycles.
- Filing multiple unrelated issues in a single report, making it impossible to close/track independently.
- Confusing severity and priority when filling in the report (see [Bug Life Cycle](../00-start-here/bug-life-cycle.md) for the distinction) — this misroutes urgency and can bury a high-severity, low-priority-looking bug.
- Not attempting to isolate the minimal reproduction steps before filing — a report that says "sometimes happens, not sure when" is far less actionable than one that's narrowed the trigger down.

## Interview Notes

- Be ready to write a well-structured defect report live from a described bug scenario — a very common practical exercise.
- Know severity vs. priority cold, with an example of when they diverge (cosmetic-but-visible bug = low severity, high priority).
- Be able to explain what evidence you'd want captured automatically from a failing automated test to make it as actionable as a well-written manual report.

## References

- [ISTQB Foundation Level Syllabus — Defect Reports](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [Atlassian — How to Write a Good Bug Report](https://www.atlassian.com/software/jira/guides/issue-types/bug)