# Monitoring & Alerting Basics for QA

## What It Is

This note covers what an SDET should understand about dashboards, alerting thresholds, and on-call practices — enough to use monitoring tools effectively during investigation and to contribute meaningfully to alert design — without taking on full SRE ownership of an organization's monitoring infrastructure. This closes the folder by connecting the metrics pillar from [Observability Fundamentals](./observability-fundamentals.md) to the operational practice of actually watching and responding to them.

## Why It Matters

- Dashboards and alerts are frequently the *first* signal that something is wrong — often before any bug report or failing test surfaces the issue — an SDET who can read and interpret existing dashboards effectively gets a head start on investigation, and one who can contribute to alert design helps ensure real regressions get caught proactively.
- This mirrors the same SDET-scope boundary established in [Security Testing Fundamentals](../13-security-testing/security-testing-fundamentals.md) — an SDET can and should meaningfully engage with monitoring/alerting, but full ownership of an organization's alerting strategy and on-call rotation is typically an SRE/platform engineering responsibility.
- Poorly calibrated alerts (too sensitive, too noisy) train teams to ignore them — the same "gate that trains bypass behavior" risk covered in [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md) applies directly to alerting, and an SDET's input on what's actually worth alerting on is valuable.

## How It Works

**What a typical monitoring dashboard shows:** the key metrics from [Observability Fundamentals](./observability-fundamentals.md) — error rate, latency percentiles, throughput — usually broken down per service, often with a time-series graph showing recent trend, letting anyone glance at it and assess "is the system currently healthy."

**Alert design basics:**
- **Threshold-based alerts** — fire when a metric crosses a defined value (error rate > 5% for 5 minutes) — simple, but can be noisy if poorly tuned (too sensitive) or miss real issues (too lenient).
- **Anomaly-based alerts** — fire based on deviation from a learned historical baseline, rather than a fixed threshold — can catch gradual or unusual patterns a fixed threshold would miss, but requires enough historical data to establish a meaningful baseline.
- **Alert fatigue** — the well-documented phenomenon where too many low-value/noisy alerts cause a team to start ignoring alerts generally, including genuinely critical ones — this is the alerting-specific version of the bypass-training risk seen with quality gates and flaky tests throughout this repo.

**Where an SDET reasonably contributes, without owning the full system:**
- Suggesting what's actually worth alerting on, based on knowledge of which flows are business-critical (connecting to [Risk-Based Testing](../00-start-here/risk-based-testing.md)).
- Using dashboards during test failure investigation (per [Observability Fundamentals](./observability-fundamentals.md)'s metric → trace → logs flow) rather than only relying on test-framework-level artifacts.
- Flagging when an alert fired for something that turned out not to be a real issue (a false positive), contributing to better calibration over time.

## Example

A dashboard-reading exercise during routine test failure investigation, showing the practical, everyday use case:

```text
An E2E smoke test just failed in CI with a checkout timeout.

Before assuming it's a flaky test (per Flaky Test Handling) or
diving into logs, a quick glance at the team's shared monitoring
dashboard shows:

  Checkout API error rate: currently ELEVATED (6.2%, normally <0.5%)
  Started: ~12 minutes ago
  Latency p95: also elevated, from 400ms baseline to 3.1s

This immediately tells us: the test failure is NOT a flaky/isolated
test issue — it's correctly reflecting a REAL, currently ongoing
production/staging incident. This reframes the investigation
entirely: instead of debugging the test, the priority becomes
investigating (or escalating) the actual system issue, using the
trace/log techniques from earlier notes in this folder.
```

A concrete example of an SDET contributing to alert calibration, illustrating the "flag false positives" contribution mentioned above:
```text
Observation: An alert for "checkout error rate > 2%" has fired 14
times in the past month, but investigation showed 11 of those were
brief, self-resolving blips (under 90 seconds) with no real user
impact — genuine noise contributing to alert fatigue.

Suggested calibration change: require the error rate to stay above
2% for at least 3 CONSECUTIVE minutes before alerting, rather than
firing on any single data point crossing the threshold — this
change, informed by QA's pattern-recognition from investigating
the false-positive incidents, reduces noise while still catching
genuinely sustained issues.
```

## Production Considerations

- Familiarize yourself with your team's actual dashboards and alert channels as part of onboarding, not as an afterthought — knowing where to look during an investigation (per the checkout example above) meaningfully speeds up triage.
- When suggesting alert calibration changes, bring concrete evidence (a pattern of false positives, as in the example) rather than a vague "this alert is too noisy" — the same evidence-based discipline from [Debugging Flaky Production Issues](./debugging-flaky-production-issues.md) applies to alert tuning too.
- Recognize the boundary of reasonable SDET involvement — contributing input on what's worth alerting on and flagging noise is valuable; owning the full alerting infrastructure, on-call rotation design, and incident response process is typically outside standard SDET scope, similar to the specialist boundary in [Security Testing Fundamentals](../13-security-testing/security-testing-fundamentals.md).

## Common Pitfalls

- Assuming a CI test failure is "just flaky" without first checking whether it correlates with a real, currently-visible dashboard anomaly — missing that the test may be correctly reflecting a genuine, ongoing issue.
- Not knowing where the team's core dashboards/alert channels are, missing an easy, fast first step during investigation.
- Suggesting alert threshold changes based on impression alone rather than gathering concrete evidence of false positives/negatives first.
- Overstepping into full ownership of alerting/monitoring infrastructure design without the SRE/platform context that typically informs those broader architectural decisions — contributing input is valuable; assuming full ownership without that context can lead to poorly informed changes.

## Interview Notes

- Be ready to describe how you'd use a monitoring dashboard as a first step when investigating a test failure — specifically, checking whether a failure correlates with a real, currently visible system anomaly before assuming it's test flakiness.
- Understand alert fatigue and why poorly calibrated alerts are actively harmful, not just annoying — connecting to the broader "erodes trust in the signal" pattern seen with flaky tests and quality gates throughout this repo.
- Be able to describe the reasonable boundary of SDET involvement in monitoring/alerting — contributing informed input versus owning the full infrastructure — showing the same scope judgment expected in [Security Testing Fundamentals](../13-security-testing/security-testing-fundamentals.md).

## References

- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PagerDuty — Alert Fatigue](https://www.pagerduty.com/resources/learn/what-is-alert-fatigue/)