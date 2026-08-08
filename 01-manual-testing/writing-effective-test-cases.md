# Writing Effective Test Cases

## What It Is

A well-written test case is a self-contained, unambiguous procedure that any tester — not just the person who wrote it — can execute and get the same result. Writing effective test cases is a distinct skill from test *design* (choosing what to test, see [Test Case Design Techniques](../00-start-here/test-case-design-techniques.md)) — this is about how to *express* a chosen test case clearly and consistently.

## Why It Matters

- A poorly written test case (missing preconditions, vague expected results) produces inconsistent execution — two testers running the "same" test case can get different results simply because the case was ambiguous.
- Test cases are a communication artifact, not just a personal reminder — they're read by other testers, new team members, developers verifying fixes, and sometimes auditors. Clarity has direct downstream cost if it's missing.
- Well-structured test cases translate cleanly into automation later — vague ones require the automation engineer to reverse-engineer intent, wasting time and risking incorrect assumptions.

## How It Works

A complete, effective test case includes these fields, at minimum:

| Field | Purpose |
|---|---|
| ID | Unique identifier for traceability (linking to requirements, defects, automation) |
| Title | Short, descriptive summary of what's being verified |
| Preconditions | State the system must be in before starting (account exists, specific data seeded, etc.) |
| Test Data | Specific input values — not placeholders like "valid email" |
| Steps | Numbered, unambiguous actions — one action per step |
| Expected Result | Specific, verifiable outcome — not vague ("it should work") |
| Priority/Severity | How critical this case is, for execution ordering under time pressure |

**Principles for clarity:**
- One test case verifies one thing. If you're using "and" to describe what a case verifies, it's probably two cases.
- Steps should be actions, not judgments — "click Submit" not "check the form works."
- Expected results should be specific and checkable — "user is redirected to /dashboard and sees welcome message" not "login works correctly."
- Use concrete test data, not placeholders — "test@example.com / Pass@123" not "a valid email and password."

## Example

**Poorly written (ambiguous, hard to execute consistently):**
```text
Test: Check login
Steps: Try logging in
Expected: Should work correctly
```

**Effectively written (specific, unambiguous, repeatable):**
```text
Test Case ID: TC_LOGIN_001
Title: Verify successful login with valid credentials
Priority: P1 (Critical path)

Preconditions:
  - User account exists: email = test@example.com, password = Pass@123
  - User is logged out and on the /login page

Test Data:
  Email: test@example.com
  Password: Pass@123

Steps:
  1. Enter "test@example.com" in the Email field
  2. Enter "Pass@123" in the Password field
  3. Click the "Login" button

Expected Result:
  - User is redirected to /dashboard within 3 seconds
  - Header displays the text "Welcome, test@example.com"
  - No error messages are shown
```

The second version can be handed to any tester, or converted directly into an automated script, with no guesswork:

```python
def test_login_success(page):
    page.goto("https://example.com/login")
    page.fill("#email", "test@example.com")
    page.fill("#password", "Pass@123")
    page.click("#login-btn")

    page.wait_for_url("**/dashboard", timeout=3000)
    assert page.get_by_text("Welcome, test@example.com").is_visible()
```

## Production Considerations

- Test cases should be stored in a shared, versioned location (test management tool like TestRail/Xray, or even a structured Markdown/CSV in the repo) — not scattered across personal notes or chat messages, or they become unmaintainable and untraceable.
- Link each test case to the requirement it verifies (an ID or reference) — this is what enables a Requirements Traceability Matrix and answers "what happens to our coverage if this requirement changes?"
- When a test case is automated, keep the manual test case reference (ID) in the automated script as a comment/tag — this preserves traceability between the design artifact and its automated implementation.

## Common Pitfalls

- Writing steps as vague intentions ("verify the form works") instead of concrete actions and checkable outcomes.
- Bundling multiple checks into a single test case, making it unclear which part failed when it does fail, and harder to automate cleanly.
- Using placeholder data ("a valid email") instead of actual values — this causes different testers to use different data and get inconsistent results, especially with edge-case-sensitive logic.
- Not specifying preconditions, causing test failures that are actually setup/environment issues rather than real defects — this wastes triage time.

## Interview Notes

- Be ready to critique a poorly written test case and rewrite it — a common practical interview exercise.
- Understand why "one test case, one verification" matters for automation, defect triage, and reporting clarity — this connects manual test case quality directly to automation quality.
- Be able to explain what makes a test case "automatable" vs. not — chiefly: concrete data, deterministic steps, and a specific, checkable expected result.

## References

- [ISTQB Foundation Level Syllabus — Test Case Development](https://www.istqb.org/certifications/certified-tester-foundation-level)