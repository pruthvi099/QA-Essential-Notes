# Test Case Design Techniques

## What It Is

Test case design techniques are systematic methods for deriving a small, high-value set of test cases from requirements — instead of guessing inputs or testing everything exhaustively (which is usually impossible). They fall into two broad categories:

- **Black-box techniques** — derived from requirements/specifications, without knowledge of internal code (Equivalence Partitioning, Boundary Value Analysis, Decision Table, State Transition).
- **White-box techniques** — derived from internal code structure (statement coverage, branch coverage, path coverage) — used more by developers writing unit tests.

This note focuses on the black-box techniques an SDET uses most often when designing functional test cases.

## Why It Matters

- Exhaustive testing is almost always impossible (a single integer input field has billions of possible values). These techniques let you find the *smallest set of test cases that gives the highest defect-finding confidence*.
- Interviewers frequently give a small spec (e.g., "an age field accepts 18–60") and ask you to design test cases — they're testing whether you apply these techniques or just guess randomly.
- These techniques directly translate into good automated test parametrization — a well-designed technique maps cleanly onto `pytest.mark.parametrize` or a data-driven test.

## How It Works

**1. Equivalence Partitioning (EP)** — divide inputs into groups (partitions) where all values in a group should behave the same way; test one representative value per partition instead of every value.

**2. Boundary Value Analysis (BVA)** — most defects occur at the edges of partitions, not in the middle. Test the boundary values and the values immediately on either side.

**3. Decision Table Testing** — used when output depends on a *combination* of multiple conditions; a table lists every meaningful combination of inputs and their expected output.

**4. State Transition Testing** — used when the system behaves differently depending on its current state (e.g., an order that's "Pending" vs "Shipped" vs "Cancelled").

### Example spec: "Age field accepts values 18 to 60 (inclusive)"

| Technique | Test Values | Reasoning |
|---|---|---|
| Equivalence Partitioning | 10 (invalid-low), 35 (valid), 75 (invalid-high) | One representative per partition |
| Boundary Value Analysis | 17, 18, 19, 59, 60, 61 | Values at and adjacent to each boundary |

## Example

Applying EP + BVA together in a parametrized Python test — this is the practical, automatable output of the technique:

```python
import pytest

def validate_age(age: int) -> bool:
    return 18 <= age <= 60

@pytest.mark.parametrize("age, expected", [
    # Equivalence partitions
    (10, False),   # invalid - below range
    (35, True),    # valid - mid range
    (75, False),   # invalid - above range
    # Boundary values
    (17, False),   # just below lower boundary
    (18, True),    # lower boundary
    (19, True),    # just above lower boundary
    (59, True),    # just below upper boundary
    (60, True),    # upper boundary
    (61, False),   # just above upper boundary
])
def test_age_validation(age, expected):
    assert validate_age(age) == expected
```

A decision table example for a combined-condition rule — "free shipping if (order > ₹500) AND (loyalty member)":

| Order > ₹500 | Loyalty Member | Free Shipping? |
|---|---|---|
| Yes | Yes | Yes |
| Yes | No | No |
| No | Yes | No |
| No | No | No |

Each row becomes one test case — this table itself *is* the test design artifact, and maps directly to four parametrized test entries.

## Production Considerations

- These techniques should be applied *before* writing automation code — designing test cases well is a separate skill from scripting them, and skipping this step is how teams end up with redundant or gap-ridden suites.
- Decision tables scale poorly with many conditions (2 conditions = 4 rows, 5 conditions = 32 rows) — for large combinations, pairwise/combinatorial testing techniques are used instead to reduce the set intelligently.
- BVA depends on knowing the *actual* boundary implementation (inclusive vs exclusive) — get this from requirements or code review, not assumption, since off-by-one boundary bugs are extremely common.

## Common Pitfalls

- Testing only "happy path" mid-range values and skipping boundaries — this is where most real defects live.
- Applying EP without BVA (or vice versa) — they're complementary; EP finds category-level gaps, BVA finds edge-level gaps.
- Building decision tables for independent (non-combinatorial) conditions, which is unnecessary — decision tables are for when conditions interact.
- Over-testing: writing a test for every single value instead of trusting the technique to have already selected a representative, sufficient set.

## Interview Notes

- Be ready to design test cases live for a given spec using EP + BVA — extremely common interview exercise.
- Know how to justify *why* a test value was chosen ("this is the upper boundary, off-by-one errors are common here") rather than just listing values.
- Be able to explain when a decision table is the right tool vs. when EP/BVA alone is sufficient (i.e., when conditions interact vs. when they're independent).

## References

- [ISTQB Foundation Level Syllabus — Test Techniques](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [ISTQB Glossary — Boundary Value Analysis](https://glossary.istqb.org/en_US/term/boundary-value-analysis)