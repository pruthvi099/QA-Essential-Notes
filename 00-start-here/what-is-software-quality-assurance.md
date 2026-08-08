# What Is Software Quality Assurance (QA)

## What It Is

Quality Assurance (QA) is the discipline of ensuring that software meets defined functional and non-functional requirements before and after it reaches users. QA is broader than "testing" — testing is the activity of executing a system to find defects; QA is the overall process (standards, reviews, testing, monitoring) that builds confidence that a product works as intended.

Three related terms are often confused:

| Term | Focus |
|---|---|
| Quality Assurance (QA) | Process-oriented — preventing defects through standards, reviews, and process improvement |
| Quality Control (QC) | Product-oriented — inspecting the actual product/build for defects |
| Testing | Execution-oriented — running the software to find bugs (a subset of QC) |

## Why It Matters

- Defects found late (in production) cost significantly more to fix than defects found early (in design or development) — this is the basis of the "shift-left" testing philosophy.
- QA is not just "clicking through the app" — it directly influences release confidence, customer trust, and business risk (financial loss, compliance failure, reputational damage).
- As an SDET, you sit at the intersection of QA and engineering: you don't just find bugs, you build the systems (automation, pipelines, tooling) that let a team find bugs faster and more reliably.

## How It Works

A typical QA process follows the Software Testing Life Cycle (STLC), which runs in parallel with the Software Development Life Cycle (SDLC):

1. **Requirement Analysis** — understand what is being built, identify testable requirements and ambiguities.
2. **Test Planning** — define scope, approach, resources, environments, entry/exit criteria.
3. **Test Case Design** — write test cases/scenarios from requirements (manual and/or automated).
4. **Environment Setup** — prepare test data, environments, and tooling.
5. **Test Execution** — run tests, log results, raise defects.
6. **Test Closure** — evaluate coverage, exit criteria, and lessons learned.

Defects found during execution follow a **bug life cycle**: New → Assigned → In Progress → Fixed → Retest → Closed (or Reopened if the retest fails).

## Example

A minimal example of what a QA engineer produces early in the STLC — a test case derived from a requirement:

**Requirement:** "Users must be able to log in with a valid email and password."

```text
Test Case ID: TC_LOGIN_001
Title: Verify successful login with valid credentials
Preconditions: User account exists with email test@example.com / password Pass@123
Steps:
  1. Navigate to /login
  2. Enter email: test@example.com
  3. Enter password: Pass@123
  4. Click "Login"
Expected Result: User is redirected to /dashboard and sees their username in the header
```

This test case is what later gets automated (see Playwright/Selenium notes) once it's stable and repeated frequently enough to justify automation.

## Production Considerations

- Not every test case should be automated. QA decides *what* to test; automation strategy decides *how much of that* should run without a human (see [Testing Strategies](../21-testing-strategies/)).
- QA processes should be lightweight enough not to slow delivery, but rigorous enough to catch risk — this balance is a constant, evolving decision, not a one-time setup.
- In mature teams, QA activities (requirement review, risk analysis) start before a single line of code is written — this is what "shift-left" means in practice.

## Common Pitfalls

- Treating QA and "manual testing" as synonymous — QA includes process, prevention, and standards, not just execution.
- Skipping requirement analysis and going straight to test execution, which causes tests to validate the wrong behavior.
- Assuming 100% test coverage means 100% quality — coverage measures what was checked, not whether the right things were checked.
- Not defining clear entry/exit criteria, leading to ambiguous "done" states for testing phases.

## Interview Notes

- Be ready to explain the difference between QA, QC, and Testing precisely — this is one of the most common opening interview questions.
- Know the STLC phases and be able to map them to SDLC phases (e.g., V-model).
- Understand the bug life cycle states and what "Reopened" vs "Rejected" vs "Deferred" mean in a defect tracker (Jira, Azure DevOps).

## References

- [ISTQB Glossary — Quality Assurance](https://glossary.istqb.org/en_US/term/quality-assurance)
- [ISTQB Foundation Level Syllabus](https://www.istqb.org/certifications/certified-tester-foundation-level)