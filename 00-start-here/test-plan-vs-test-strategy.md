# Test Plan vs. Test Strategy

## What It Is

A **test strategy** is a high-level, organization- or project-wide document defining the general approach to testing — what types/levels of testing will be used, tools, environments, and overall risk approach. It rarely changes and is usually written once per project or organization.

A **test plan** is a more detailed, often release- or feature-specific document that defines scope, schedule, resources, entry/exit criteria, and deliverables for a *specific* testing effort. It's derived from the strategy and changes more frequently.

| | Test Strategy | Test Plan |
|---|---|---|
| Scope | Organization/project-wide | Specific release/feature/sprint |
| Frequency of change | Rare | Frequent |
| Owner | QA Lead / Test Architect | Test Lead / SDET |
| Content | Approach, tools, standards | Scope, schedule, resources, criteria |

## Why It Matters

- These are two of the most commonly confused terms in QA vocabulary, and interviewers use them to check whether you understand testing as a planned engineering activity rather than ad-hoc execution.
- As an SDET, you'll rarely write the strategy document, but you'll frequently work within one — and you will write test plans for specific features/releases, so you need to know what belongs in each.
- A clear entry/exit criteria section (part of a test plan) is what actually determines whether a release is "ready" — vague or missing criteria is a common real-world cause of shipping under-tested software.

## How It Works

**A test strategy typically defines:**
- Overall testing approach (risk-based, requirement-based, etc.)
- Levels of testing to be performed and by whom (dev vs QA)
- Types of testing in scope (functional, performance, security, etc.)
- Tools and frameworks standardized across the org
- Defect management process and severity/priority definitions
- Automation approach (what gets automated, and general pyramid shape)

**A test plan typically defines (per IEEE 829 / ISTQB structure):**
- Scope — what's being tested (and explicitly, what's NOT)
- Test items — features/modules under test
- Approach — techniques used for this specific effort
- Entry criteria — conditions that must be true before testing starts
- Exit criteria — conditions that must be true to consider testing complete
- Schedule and resources
- Risks and contingencies
- Deliverables (test reports, defect logs, coverage reports)

## Example

A condensed real-world test plan excerpt for a single feature release:

```text
Test Plan: Checkout Discount Code Feature — Release 4.2

Scope:
  In scope: Discount code entry, validation, and application at checkout
  Out of scope: Loyalty points system (separate feature, tested independently)

Test Items:
  - Discount code input field (UI)
  - POST /api/checkout/apply-discount (API)
  - Discount calculation logic (unit-level, owned by dev team)

Approach:
  - Unit tests: written by developers, run on every commit
  - API tests: automated (pytest + requests), run on every PR
  - E2E tests: automated (Playwright), run nightly
  - Exploratory manual testing: 1 day, focused on edge-case codes and UX

Entry Criteria:
  - Feature deployed to staging
  - Unit test coverage ≥ 80% on new code
  - No open Blocker/Critical defects from code review

Exit Criteria:
  - All planned automated tests passing
  - No open Critical/High defects
  - Manual exploratory session completed with findings triaged

Resources: 1 SDET (automation), 1 QA Engineer (exploratory)
Schedule: 3 working days, aligned to sprint end
Risks: Discount service has an external dependency (promo-code provider);
       mitigated by mocking it in API tests, manual smoke test against real
       provider before release
```

## Production Considerations

- Entry/exit criteria should be specific and measurable ("all P1 automated tests pass," not "testing looks good") — vague criteria are unenforceable and get overridden under release pressure.
- In fast-moving Agile teams, full IEEE 829-style test plans are often replaced by a lightweight test plan section inside the user story or a shared team wiki — but the *content* (scope, approach, criteria) should still exist