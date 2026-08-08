# Bug (Defect) Life Cycle

## What It Is

The bug life cycle (defect life cycle) is the sequence of states a defect moves through from the moment it's discovered to the moment it's resolved and verified. It gives every defect a tracked, unambiguous status so testers, developers, and managers agree on what's outstanding and what's done.

## Why It Matters

- Without a defined life cycle, defects get lost, duplicated, or closed without verification — directly hurting release quality.
- SDETs interact with defect tracking constantly: raising bugs from automated test failures, triaging flaky-vs-real failures, and verifying fixes. Understanding the states precisely is a daily-use skill, not just interview trivia.
- The states you choose (and how strictly you enforce transitions) reflect real process maturity — this is commonly discussed in interviews to gauge whether you've worked in a real defect-management workflow (Jira, Azure DevOps, etc.).

## How It Works

A typical bug life cycle:

```text
New → Assigned → In Progress → Fixed → Retest → Closed
                     ↓                     ↓
                  Rejected             Reopened (if retest fails)
                     ↓
                  Deferred
```

| State | Meaning |
|---|---|
| New | Defect logged, not yet triaged |
| Assigned | Triaged and assigned to a developer |
| In Progress | Developer actively working the fix |
| Fixed | Developer believes the issue is resolved, ready for retest |
| Retest | Tester re-executes the original steps against the fix |
| Closed | Retest passed; defect confirmed resolved |
| Reopened | Retest failed; defect goes back to In Progress |
| Rejected | Not a valid defect (e.g., working as intended, duplicate, or not reproducible) |
| Deferred | Valid defect, but fix postponed to a future release (business decision) |

Some teams add **Duplicate** (already logged elsewhere) as a distinct terminal state from Rejected, since they have different root causes and different reporting implications.

## Example

A well-written defect report — the artifact that drives this whole life cycle — needs enough detail for a developer to reproduce the issue without back-and-forth:

```text
Title: Checkout fails with 500 error when cart contains 0-price item

Severity: High       Priority: P1
Environment: Staging, Chrome 126, macOS

Steps to Reproduce:
  1. Add a promotional (₹0) item to cart
  2. Add a paid item to cart
  3. Proceed to checkout
  4. Click "Place Order"

Expected Result: Order is placed successfully
Actual Result: API returns 500 Internal Server Error
                (see attached response body and screenshot)

Additional Info:
  - Reproducible 3/3 times
  - Does NOT occur if the ₹0 item is removed
  - API log excerpt: NullReferenceException in OrderTotalCalculator.cs line 42
```

When this comes from an automated test failure, the same detail should be captured automatically — screenshot, trace, request/response, and logs attached to the CI failure — so triage doesn't require manually reproducing it.

## Production Considerations

- Automated test failures should not be auto-filed as bugs without triage — flaky failures (network blips, timing issues) pollute the tracker and erode trust in automation. Most teams separate "test failed" from "confirmed defect."
- Severity (impact on the system) and Priority (urgency to fix) are independent axes and should be set independently — a cosmetic bug on the homepage can be high priority (visible to every user) but low severity.
- Reopen rate is a useful team health metric: a high reopen rate signals either poor reproduction steps, poor fix verification, or unstable environments.

## Common Pitfalls

- Closing a defect without retesting the actual fix — "developer says it's fixed" is not verification.
- Conflating severity and priority, leading to miscommunication about what should be fixed first.
- Not distinguishing "Rejected" (not a bug) from "Deferred" (valid bug, fix postponed) — these have very different implications for release risk.
- Filing vague defect reports ("checkout doesn't work") without reproducible steps, environment, and evidence — this is the single biggest cause of slow defect resolution.

## Interview Notes

- Be ready to draw the full life cycle from memory, including Rejected/Deferred/Reopened branches, not just the happy path.
- Know the difference between severity and priority with an example of when they diverge.
- Be able to describe how you'd triage an automated test failure before deciding whether it becomes a filed defect (a very common SDET-specific interview question).

## References

- [ISTQB Foundation Level Syllabus — Defect Management](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [Atlassian — Bug Life Cycle in Jira](https://www.atlassian.com/software/jira/guides/issue-types/bug)