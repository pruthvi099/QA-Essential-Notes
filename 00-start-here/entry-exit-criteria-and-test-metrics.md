# Entry/Exit Criteria & Test Metrics

## What It Is

**Entry criteria** are the conditions that must be true *before* a testing phase can begin. **Exit criteria** are the conditions that must be true to consider that phase *complete*. **Test metrics** are the quantitative measurements used to objectively evaluate whether exit criteria have actually been met, and to track quality trends over time.

Together, these turn "is testing done?" and "is the release ready?" from subjective opinions into answerable, evidence-based questions.

## Why It Matters

- Without explicit entry/exit criteria, testing has no defined start or end point — teams either test forever (never confident enough to stop) or stop arbitrarily under release pressure (shipping under-tested software).
- Metrics are what make exit criteria enforceable and auditable — "exit criteria: no critical defects open" is only meaningful if defect counts are actually tracked and reported.
- SDETs are often responsible for producing these metrics from automated test runs (CI reports, coverage tools) — this is a direct, practical extension of automation work, not just a manual/process topic.

## How It Works

**Entry criteria** — typically checked before test execution starts:
- Build/feature deployed to the target test environment
- Test environment stable and accessible
- Test data prepared
- Required test cases/scripts ready
- No blocker-level defects preventing test execution (e.g., app doesn't launch)

**Exit criteria** — typically checked before declaring testing complete:
- Defined % of planned test cases executed (e.g., 100% of P1 cases, ≥90% overall)
- No open Critical/Blocker defects; agreed threshold for lower-severity defects
- Defined code/requirement coverage achieved
- All planned regression tests passing in CI

**Common test metrics:**

| Metric | Formula / Meaning |
|---|---|
| Test Case Pass Rate | (Passed cases / Total executed cases) × 100 |
| Defect Density | Defects found / size of the module (e.g., per 1,000 lines of code, or per feature) |
| Requirement Coverage | Requirements with at least one test case / Total requirements |
| Code Coverage | Lines/branches executed by tests / Total lines/branches (from unit test tooling) |
| Defect Leakage | Defects found in production / Total defects found (pre + post release) — measures how many bugs testing missed |
| Test Execution Progress | Test cases executed so far / Total planned test cases |

## Example

A concrete entry/exit criteria + metrics section, as it would appear in a real test plan (see [Test Plan vs. Test Strategy](./test-plan-vs-test-strategy.md)):

```text
Entry Criteria:
  - Build deployed to staging, smoke test passing
  - Test data seeded (10 test user accounts, 3 product categories)
  - No open Blocker defects from code review

Exit Criteria:
  - 100% of P1 (critical path) test cases executed and passing
  - ≥ 95% of all planned test cases executed
  - Zero open Critical/Blocker defects
  - ≤ 3 open Medium-severity defects, each with a documented workaround
  - Regression suite green in CI (last run within 24 hours of release)

Metrics Snapshot (end of test cycle):
  Test Case Pass Rate:     97.2% (105/108 executed)
  Requirement Coverage:    100% (22/22 requirements have ≥1 test case)
  Open Defects:            0 Critical, 1 High (deferred, approved), 2 Medium
  Defect Leakage (prior release): 8% (2 of 25 defects found post-release)
```

A CI pipeline can compute several of these automatically from test run results:

```python
# Simple pass-rate calculation from a pytest JSON report, usable in a CI gate
import json

def get_pass_rate(report_path: str) -> float:
    with open(report_path) as f:
        report = json.load(f)

    total = report["summary"]["total"]
    passed = report["summary"].get("passed", 0)
    return round((passed / total) * 100, 1) if total else 0.0

pass_rate = get_pass_rate("pytest_report.json")
if pass_rate < 95.0:
    raise SystemExit(f"Exit criteria not met: pass rate {pass_rate}% < 95%")
```

## Production Considerations

- Exit criteria should be agreed upon *before* the testing cycle starts (during test planning), not negotiated after results come in — deciding criteria post-hoc defeats their purpose as an objective gate.
- Metrics like defect leakage require tracking defects found in production, not just pre-release — this needs a feedback loop between production incident tracking and QA metrics, which many teams skip.
- Code coverage is a useful signal but a dangerous target on its own — 100% coverage with weak assertions is possible and gives false confidence; pair it with mutation testing or defect-based metrics for a fuller picture.

## Common Pitfalls

- Setting exit criteria so loose they're always met ("most tests pass") — this removes their value as an actual gate.
- Using a single metric (usually pass rate or code coverage) as the sole release decision factor, ignoring severity/impact of what's still open.
- Not tracking defect leakage at all — without it, a team can't tell whether their testing process is actually improving release quality over time.
- Treating entry criteria as optional under time pressure — starting testing against an unstable build or incomplete test data wastes the whole cycle and produces unreliable results.

## Interview Notes

- Be ready to state clear entry and exit criteria for a hypothetical release on the spot — a frequent practical interview question.
- Know defect leakage specifically — it's one of the metrics that most directly answers "is our testing effective," and is less commonly known than pass rate/coverage.
- Be able to explain why code coverage alone is an insufficient quality metric — this is a common follow-up to "what metrics do you track."

## References

- [ISTQB Foundation Level Syllabus — Test Monitoring and Control](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [IEEE 829 Standard for Software Test Documentation](https://ieeexplore.ieee.org/document/1795035)