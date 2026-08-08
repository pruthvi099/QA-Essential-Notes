# Test Documentation Overview

## What It Is

Test documentation is the set of artifacts produced during testing — distinct from each other in purpose, detail level, and audience. The terms below are often used interchangeably in casual conversation, but they mean specific, different things:

| Artifact | What It Is |
|---|---|
| Test Scenario | A one-line, high-level statement of *what* to verify — not how |
| Test Case | A detailed, step-by-step procedure with specific inputs and expected results |
| Test Script | The automated (code) implementation of a test case |
| Checklist | A quick, non-detailed list of things to verify — no fixed steps or expected results per item |
| Test Data | The specific input values used to execute a test case |
| Test Report | A summary of test execution results — pass/fail counts, defects found, coverage |

## Why It Matters

- Using the wrong artifact for the wrong purpose wastes effort — writing full step-by-step test cases for a fast smoke check, or relying on a vague checklist for a complex regulated financial flow, are both mismatches.
- SDETs move between these artifacts constantly: a test scenario becomes a test case during design, which becomes a test script during automation, executed against test data, producing a test report — understanding the chain matters for traceability (can you trace a production bug back to a missing scenario?).
- Interviewers use precise terminology here to gauge testing maturity — conflating "test case" and "test script," for example, is a common tell that someone hasn't worked in a structured testing process.

## How It Works

The typical flow from idea to executed result:

```text
Requirement
   ↓
Test Scenario   ("Verify login with valid credentials")
   ↓
Test Case       (detailed steps + specific data + expected result)
   ↓
Test Script     (automated code implementing the test case)
   ↓
Test Execution  (run against test data, produces pass/fail)
   ↓
Test Report     (aggregated results, defects, coverage)
```

A **checklist** sits outside this chain — it's used for fast, repeatable verification (e.g., pre-release sanity checks, code review checklists) where full step-by-step detail isn't needed and speed matters more than rigor.

## Example

The same requirement, shown as each artifact type, to make the distinctions concrete:

**Test Scenario** (high-level, no detail):
```text
Verify that a user cannot log in with an incorrect password.
```

**Test Case** (detailed, from the scenario above):
```text
Test Case ID: TC_LOGIN_002
Title: Verify login fails with incorrect password
Preconditions: User account exists (test@example.com / correct password: Pass@123)
Test Data: email = test@example.com, password = WrongPass1
Steps:
  1. Navigate to /login
  2. Enter email: test@example.com
  3. Enter password: WrongPass1
  4. Click "Login"
Expected Result: Login is rejected; error message "Invalid email or password" is shown;
                 user remains on /login
```

**Test Script** (the test case automated):
```python
def test_login_fails_with_incorrect_password(page):
    page.goto("https://example.com/login")
    page.fill("#email", "test@example.com")
    page.fill("#password", "WrongPass1")
    page.click("#login-btn")

    assert page.url.endswith("/login")
    assert page.get_by_text("Invalid email or password").is_visible()
```

**Checklist** (a different artifact entirely — used for a release sanity pass, not one specific flow):
```text
Pre-Release Sanity Checklist:
[ ] Homepage loads without console errors
[ ] Login works with valid credentials
[ ] Login rejects invalid credentials
[ ] Checkout completes with a test card
[ ] No broken images on primary landing pages
```

## Production Considerations

- Traceability matters: mature teams link test cases back to requirements (via IDs or a requirements traceability matrix) so coverage gaps are visible — "which requirements have no associated test case?" should be an answerable question.
- Not every scenario needs a fully detailed test case before automation — for simple, well-understood flows, some teams automate directly from the scenario level and let the test script itself serve as the detailed record (the code *is* the documentation).
- Test reports should be structured enough to be consumed by both humans (release go/no-go decisions) and dashboards/CI (pass/fail gates) — a report that's just a wall of text in a chat message doesn't scale.

## Common Pitfalls

- Using "test case" and "test script" interchangeably — a test case is the design artifact (can be manual or the basis for automation); a test script is specifically the automated code.
- Writing test cases so vague they're effectively checklists (no specific data, no specific expected result) — this causes inconsistent execution between different testers.
- Losing traceability between requirements and test cases as a project grows, making it impossible to answer "did we test this requirement?" with confidence.
- Treating a checklist as a substitute for detailed test cases on complex/high-risk flows — checklists trade rigor for speed, which is only appropriate for low-risk, fast checks.

## Interview Notes

- Be ready to define test scenario, test case, and test script distinctly and explain how they relate in sequence — commonly asked directly.
- Know what a Requirements Traceability Matrix (RTM) is and why it matters, even if you haven't built one formally.
- Be able to explain when a checklist is appropriate vs. when full test cases are needed — this signals judgment about rigor vs. speed trade-offs.

## References

- [ISTQB Foundation Level Syllabus — Test Documentation](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [IEEE 829 Standard for Software Test Documentation](https://ieeexplore.ieee.org/document/1795035)