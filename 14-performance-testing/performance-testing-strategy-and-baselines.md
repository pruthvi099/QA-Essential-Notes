# Performance Testing Strategy & Baselines

## What It Is

This note covers establishing performance baselines — a documented, agreed-upon "what does acceptable performance look like" — and deciding *when* performance testing actually belongs in a project's workflow, synthesizing the test types, tooling, bottleneck diagnosis, and CI integration from earlier notes in this folder into an overall strategic approach. This closes the folder the same way [Choosing a Test Framework From Scratch](../10-test-framework-design/choosing-a-test-framework-from-scratch.md) closed the framework design folder — pulling individual pieces into one deliberate, contextual decision.

## Why It Matters

- Without a documented baseline, "is this fast enough" is a subjective, disputable judgment call — a clear, agreed-upon baseline turns performance evaluation into an objective, testable criterion, the same principle as [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md) applied specifically to performance.
- Not every project needs the full suite of load/stress/spike/soak testing from day one — deciding *when* performance testing effort is actually warranted (and how much) is a genuine, risk-based judgment call, mirroring [Risk-Based Testing](../00-start-here/risk-based-testing.md)'s general prioritization framework.
- This is commonly asked as a capstone-style interview question specifically because it requires synthesizing multiple concepts (test types, tooling, CI integration, bottleneck diagnosis) into one coherent, practical strategy — not just reciting definitions.

## How It Works

**Establishing a performance baseline:**
1. Measure current, real production performance (via monitoring/APM tools) for key user-facing flows — this is the honest starting point, not an aspirational target.
2. Define acceptable thresholds collaboratively with stakeholders (product, engineering) — informed by real user expectations and business impact, not arbitrary round numbers.
3. Document the baseline explicitly (e.g., "checkout p95 latency must stay under 2s at 200 concurrent users") so it's a shared, referenceable standard, not tribal knowledge.
4. Revisit the baseline periodically as the application, infrastructure, and user base evolve — a baseline set a year ago may no longer reflect current realistic expectations or current infrastructure capacity.

**Deciding when performance testing is warranted (a risk-based framework, mirroring [Risk-Based Testing](../00-start-here/risk-based-testing.md)):**
- **High priority**: revenue-critical flows (checkout, payment), flows with known past performance incidents, flows expecting significant traffic growth.
- **Medium priority**: frequently-used but non-critical flows.
- **Lower priority**: rarely-used, low-traffic, or purely internal/admin flows — not zero priority, but proportionally less testing investment.

## Example

A documented performance baseline and testing strategy for a hypothetical e-commerce application, synthesizing the folder's concepts into one coherent plan:

```text
Performance Testing Strategy — E-Commerce Platform

Baselines (established from 3 months of production monitoring data):
  Checkout:        p95 < 2.0s at 200 concurrent users (current peak)
  Product search:  p95 < 800ms at 500 concurrent users
  Product listing: p95 < 500ms at 500 concurrent users

Risk-based test type allocation:
  Checkout (revenue-critical, past incident history):
    - Load test: every release
    - Stress test: quarterly, to track headroom as traffic grows
    - Spike test: before known high-traffic events (sales campaigns)
    - Soak test: quarterly

  Product search/listing (high-traffic, not directly revenue-blocking):
    - Load test: every release
    - Stress/soak: half-yearly

  Admin/internal reporting pages (low traffic, non-critical):
    - Load test: only if a specific concern arises; not routinely scheduled

CI integration (per Performance Testing in CI):
  - Lightweight smoke-level check on every PR (loose threshold)
  - Full load test suite nightly, with trend tracking
  - Full stress/spike/soak suite runs per the cadence above, not
    tied to every code change

Baseline review cadence: Quarterly, or immediately after any major
infrastructure change (e.g., a database migration, a new caching layer)
```

## Production Considerations

- Baselines should be derived from real monitoring data wherever possible, not guessed — a baseline based on incorrect assumptions about actual usage patterns produces tests that don't validate anything meaningful about real-world readiness.
- Revisit baselines after any significant infrastructure or architecture change (a database migration, see [Database Migration Testing](../05-sql-database-testing/database-migration-testing.md); a caching layer added) — an outdated baseline can either be too lenient (missing a real regression) or too strict (blocking releases over a target that's no longer realistic given new infrastructure).
- Communicate the performance testing strategy and baselines to the broader engineering team, not just QA/SDET — performance is a shared responsibility, and developers benefit from knowing the concrete targets their code is expected to meet.

## Common Pitfalls

- Setting performance baselines arbitrarily (a round number that "sounds reasonable") instead of deriving them from actual production data and real business/user impact considerations.
- Applying the same testing depth (full load/stress/spike/soak suite) uniformly across every feature regardless of actual risk/criticality, wasting testing effort on low-risk areas while potentially under-investing in genuinely critical ones.
- Setting a baseline once and never revisiting it, even after significant infrastructure changes that would materially affect what's actually achievable or expected.
- Treating performance testing as either "not needed at all" or "needed comprehensively everywhere" — the actual right answer is nearly always a deliberate, risk-based middle ground, mirroring the same judgment call from [Risk-Based Testing](../00-start-here/risk-based-testing.md).

## Interview Notes

- Be ready to describe how you'd establish a performance baseline for a hypothetical application — deriving it from real data, defining it collaboratively, documenting it clearly — a common, practical capstone-style question for this folder's material.
- Understand and be able to apply risk-based thinking to decide which features warrant full performance testing investment versus lighter coverage, mirroring [Risk-Based Testing](../00-start-here/risk-based-testing.md)'s general framework.
- Be able to synthesize the folder's individual concepts (test types, tooling, bottleneck diagnosis, CI integration) into one coherent strategy when asked an open-ended "how would you approach performance testing for this application" question — this is what interviewers are really listening for at this level.

## References

- [k6 — Testing Guides](https://k6.io/docs/testing-guides/)
- [ISTQB Foundation Level Syllabus — Performance Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)