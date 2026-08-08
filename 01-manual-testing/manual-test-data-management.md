# Manual Test Data Management

## What It Is

Test data management is the practice of preparing, organizing, isolating, and resetting the data needed to execute test cases reliably. In a manual testing context, this means having the right accounts, records, and system states available *before* a tester starts executing — without which even a well-written test case (see [Writing Effective Test Cases](./writing-effective-test-cases.md)) can't be run consistently.

## Why It Matters

- A large share of "flaky" or inconsistent manual test results trace back to bad test data, not bad test cases — a test case can be perfectly written and still fail unpredictably if the underlying data isn't isolated or reset properly between runs.
- Shared, uncontrolled test environments (where every tester uses the same accounts/data) cause tests to interfere with each other — one tester's test run silently breaks another's precondition.
- As an SDET, test data strategy is one of the concepts that carries directly into automation — the same data isolation problems (and solutions) show up in automated test frameworks, just implemented in code instead of manual setup steps.

## How It Works

**Core principles:**
- **Isolation** — each test (or tester) should use data that won't be affected by other tests running concurrently. Shared "test1@example.com" accounts used by the whole team are a common source of flaky results.
- **Repeatability** — data should be resettable to a known state before each test cycle, so the same test case produces the same result every time it's run.
- **Realism** — test data should resemble production data closely enough to catch real issues (realistic string lengths, valid-format-but-fake values), without containing actual production/PII data.
- **Traceability** — know which data set a given test run used, so failures can be reproduced and investigated.

**Common approaches:**
- **Dedicated test accounts per tester** — avoids collisions between people testing the same flow simultaneously.
- **Database seeding/reset scripts** — a known baseline data set is loaded before each test cycle, rather than relying on whatever state the environment happens to be in.
- **Data generation utilities** — scripts/tools that generate realistic but synthetic data (names, addresses, order histories) on demand, rather than manually creating records by hand each time.
- **Environment refresh cadence** — deciding how often (nightly, per-release) a shared test environment gets reset to a clean baseline.

## Example

A test data plan for a manual test cycle on an order-management feature — the kind of artifact that should exist alongside the test cases themselves:

```text
Test Data Plan — Order Management Feature (Release 4.2)

Test Accounts:
  - qa_tester1@example.com / TestPass@123 — standard customer, no orders
  - qa_tester2@example.com / TestPass@123 — standard customer, 5 past orders
  - qa_admin@example.com / TestPass@123 — admin role, for order-management screens

Seed Data (loaded via reset script before testing begins):
  - 3 product categories, 10 products with varying stock levels
    (including 1 out-of-stock item, for negative-path testing)
  - 1 active discount code: SAVE10 (10%, min order ₹500, expires in 30 days)
  - 1 expired discount code: OLDCODE (for negative-path testing)

Reset Procedure:
  Run `scripts/reset_test_data.sql` against the staging DB before each
  test cycle. Do NOT reuse data from a previous cycle without reset —
  order counts and stock levels will be inconsistent.

Data NOT to use:
  - Production data (even anonymized) is not permitted in staging per
    data policy — use only the synthetic accounts/products above.
```

A simple data-generation helper, the kind of lightweight tool that removes the need to manually create test records by hand each cycle:

```python
import random
import string

def generate_test_user(prefix="qa_test"):
    suffix = "".join(random.choices(string.ascii_lowercase + string.digits, k=6))
    return {
        "email": f"{prefix}_{suffix}@example.com",
        "password": "TestPass@123",
    }

# Generates a fresh, isolated account on demand — avoids collisions
# between testers reusing the same hardcoded test account
new_user = generate_test_user()
```

## Production Considerations

- Never use real production/PII data in test environments, even "just this once" — beyond the compliance risk, it also means test data isn't representative of clean, known states, undermining repeatability.
- Shared staging environments need an agreed reset cadence and ownership — without it, testers unknowingly build on top of each other's leftover data, and failures become hard to reproduce.
- Test data setup that's manual and slow (a tester creating 10 accounts by hand every cycle) doesn't scale — investing in seed scripts/generation utilities pays off quickly once test cycles are frequent (every sprint, every release).

## Common Pitfalls

- Reusing the same shared test account across the whole team, causing test interference that looks like flaky application behavior but is actually a data isolation problem.
- Not resetting environment state between test cycles, so old data (previous test's leftover orders, modified stock levels) silently invalidates preconditions for new test cases.
- Using unrealistic data (e.g., always testing with "Test User" / "test@test.com") that doesn't surface real issues with, say, long names, special characters, or realistic address formats.
- Accidentally testing against or copying production data into a lower environment — a compliance and security risk, not just a testing hygiene issue.

## Interview Notes

- Be ready to explain how you'd diagnose a "flaky" manual test result that turns out to be a data isolation issue rather than an application defect.
- Understand the difference between test data *generation* (creating new data on demand) and test data *seeding* (loading a known baseline before a cycle) — both are used, for different purposes.
- Be able to connect this topic to automation: the same isolation/reset principles apply in automated frameworks via fixtures and API-based setup, just executed in code (see fixtures in the automation folder once covered).

## References

- [ISTQB Foundation Level Syllabus — Test Environment](https://www.istqb.org/certifications/certified-tester-foundation-level)