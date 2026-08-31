# Performance Testing Fundamentals

## What It Is

Performance testing evaluates how a system behaves under load — its speed, responsiveness, and stability as usage increases. This note covers the core test types (load, stress, spike, soak) and the fundamental metrics (latency, throughput, error rate) that every other note in this folder builds on. This is distinct from [Mobile App Performance Testing](../06-mobile-testing/mobile-app-performance-testing.md), which covers client-side, on-device metrics — this folder focuses on backend/server-side performance under concurrent load.

## Why It Matters

- A feature that works correctly for one user can fail entirely under realistic concurrent load — performance issues are frequently invisible in normal functional testing (see [Levels of Testing](../00-start-here/levels-of-testing.md)) since a single test run rarely exercises genuine concurrency.
- Different performance test types answer genuinely different questions — running only a load test and assuming it also validates stability under sustained use (which is what a soak test checks) is a common, consequential gap.
- Performance problems discovered in production are far more costly and stressful to diagnose than ones caught in dedicated testing — this is a direct, practical extension of the "catch it early" principle that runs throughout this repo (see [SDLC Models](../00-start-here/sdlc-models.md) on shift-left testing).

## How It Works

**The four core performance test types:**

| Type | What It Tests | Typical Question Answered |
|---|---|---|
| **Load Testing** | Behavior under expected, realistic concurrent usage | "Does the system handle normal peak traffic correctly?" |
| **Stress Testing** | Behavior beyond expected capacity, pushed until it breaks | "What's the actual breaking point, and does it fail gracefully?" |
| **Spike Testing** | Behavior under a sudden, sharp increase in load | "Can the system handle a flash-sale-style traffic surge?" |
| **Soak Testing** (endurance) | Behavior under sustained load over an extended period | "Does performance degrade over time (memory leaks, resource exhaustion)?" |

**Core metrics every performance test measures:**
- **Latency (response time)** — how long a single request takes, typically measured as percentiles (p50/median, p95, p99) rather than just an average, since averages hide the slow outliers that matter most to real user experience.
- **Throughput** — how many requests/transactions the system processes per unit time (requests per second).
- **Error rate** — the percentage of requests failing (timeouts, 5xx errors) under load — a system that's "fast" but increasingly failing under load isn't actually performing well.
- **Resource utilization** — CPU, memory, database connections — helps distinguish *where* a bottleneck actually lives (see [Identifying Performance Bottlenecks](./identifying-performance-bottlenecks.md)).

**Why percentiles matter more than averages:** an average response time of 200ms can hide a p99 of 5 seconds — meaning 1% of users have a genuinely terrible experience while the average looks fine. Since that 1% at real scale can be thousands of actual users, p95/p99 are usually the metrics that matter most for user-facing performance.

## Example

A performance test plan distinguishing the four types for a single feature — checkout — showing how each answers a different question:

```text
Feature: Checkout

Load Test: Simulate 200 concurrent users completing checkout,
  matching typical peak-hour traffic from analytics.
  Success criteria: p95 latency < 2s, error rate < 0.1%

Stress Test: Gradually increase concurrent users (200 → 500 → 1000 →
  2000...) until the system's error rate exceeds 5% or latency
  becomes unacceptable.
  Goal: identify the actual breaking point and HOW it fails —
  does it degrade gracefully (slower but still correct) or fail
  catastrophically (500 errors, data corruption)?

Spike Test: Jump from 50 to 2000 concurrent users within 30 seconds,
  simulating a flash sale or marketing campaign traffic surge.
  Success criteria: system recovers to normal performance within
  2 minutes of the spike subsiding, no requests lost/corrupted
  during the spike itself.

Soak Test: Sustain 300 concurrent users continuously for 8 hours.
  Success criteria: p95 latency and error rate remain STABLE
  throughout — a gradual increase over the 8 hours would indicate
  a memory leak or connection pool exhaustion, not caught by a
  short load test.
```

A simple metric summary table from an actual test run, showing why percentiles are reported alongside the average:
```text
Load Test Results — Checkout Endpoint (200 concurrent users, 10 min)

Average latency:  340ms
p50 (median):     280ms
p95:               890ms
p99:              2,400ms   ← 1% of requests take over 2 SECONDS,
                               invisible if only the average were reported
Throughput:        580 req/s
Error rate:        0.3%
```

## Production Considerations

- Base load test traffic patterns on real production analytics (actual peak concurrent users, actual request mix) rather than arbitrary round numbers — a load test simulating unrealistic traffic patterns produces results that don't actually predict real-world behavior.
- Stress testing should be run in an isolated environment that can't affect real users or production data — pushing a system deliberately past its breaking point is exactly the kind of activity that needs a safe, isolated environment (see [Docker Compose for Test Stacks](../09-docker/docker-compose-for-test-stacks.md) for spinning up an isolated test stack).
- Soak testing requires meaningful time investment (hours, not minutes) — this is a real scheduling/resource cost worth planning for deliberately, not something that fits into a quick pre-release check the way a short load test might.

## Common Pitfalls

- Running only a load test and treating it as if it validates stability or breaking-point behavior too, when those require the distinct stress/soak test types specifically designed to answer those different questions.
- Reporting only average latency, hiding a poor p95/p99 experience affecting a meaningful fraction of real users.
- Testing with unrealistic traffic patterns (arbitrary user counts, unrealistic request mixes) that don't reflect actual production usage, producing results that don't predict real behavior.
- Treating "the system didn't crash" as sufficient success criteria without checking error rate and latency degradation — a system can technically stay "up" while performing unacceptably badly under load.

## Interview Notes

- Be ready to distinguish load, stress, spike, and soak testing precisely, and give an example of what each would specifically catch that the others wouldn't — a common, foundational performance testing question.
- Understand why percentiles (p95/p99) matter more than averages for evaluating real user experience, and be able to explain the "hidden outlier" problem with averages concretely.
- Be able to describe how you'd derive realistic load test traffic patterns from actual production data rather than guessing.

## References

- [k6 — Types of Load Testing](https://k6.io/docs/test-types/introduction/)
- [ISTQB Foundation Level Syllabus — Performance Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)