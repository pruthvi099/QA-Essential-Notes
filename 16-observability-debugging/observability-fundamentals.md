# Observability Fundamentals

## What It Is

Observability is the ability to understand a system's internal state from its external outputs — traditionally broken into three pillars: **logs** (discrete, timestamped event records), **metrics** (numeric measurements aggregated over time), and **traces** (the path a single request takes across a system's components). This note establishes the framework the rest of this folder builds on — when to reach for each pillar during investigation.

## Why It Matters

- This directly extends [Logging Strategy for Test Frameworks](../10-test-framework-design/logging-strategy-for-test-frameworks.md)'s framework-level logging into the *application's own* observability — when a test fails against a real, deployed system, the application's logs/metrics/traces (not just the test framework's own logs) are often what's needed to actually diagnose the root cause.
- Different pillars answer fundamentally different questions — a metric tells you *that* something is wrong (error rate spiked at 2pm), a trace tells you *where* in a multi-service request that failure occurred, and a log tells you *why* (the specific error message/stack trace) — reaching for the wrong pillar first wastes investigation time.
- As applications increasingly use microservices/distributed architectures, an SDET's debugging skill set needs to extend beyond the test framework's own artifacts (see [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md)) into the application's actual observability stack — this is a genuine, growing expectation in modern SDET roles.

## How It Works

**Logs** — discrete events with a timestamp and message, optionally structured (JSON) with additional fields (user ID, request ID, severity). Best for: understanding exactly what happened at a specific point, including error details and stack traces (see [Reading Application Logs for Debugging](./reading-application-logs-for-debugging.md)).

**Metrics** — numeric values aggregated over time (request count, error rate, latency percentiles — see [Performance Testing Fundamentals](../14-performance-testing/performance-testing-fundamentals.md) for the same percentile concepts applied here). Best for: spotting *that* something changed or is currently abnormal, and *when* it started, across an entire system rather than one specific event.

**Traces** — represent the path a single request takes as it moves through multiple services/components, broken into **spans** (individual units of work within the trace, e.g., "database query," "call to payment service"). Best for: understanding *where* in a multi-step, multi-service flow a failure or slowdown occurred (see [Distributed Tracing Basics](./distributed-tracing-basics.md)).

**The typical investigative flow, moving from broad to specific:**
1. A metric alert or dashboard shows something is wrong (elevated error rate, latency spike).
2. A trace for a specific failing request shows *which service/step* in the flow is failing or slow.
3. Logs from that specific service, filtered to the relevant time window/request ID, show the *exact error* causing the failure.

## Example

A realistic investigation showing all three pillars used together, in the order they're typically consulted:

```text
Step 1 — METRIC: A dashboard shows the checkout API's error rate
jumped from 0.2% to 8% starting at 14:32 UTC. This tells us
SOMETHING broke and roughly WHEN, but not what or where specifically.

Step 2 — TRACE: Looking at a sample of failing request traces from
that time window, each shows the failure consistently occurring at
the "payment-service" span, not in the "order-service" or
"inventory-service" spans that precede it. This narrows WHERE the
problem lives — the payment service specifically, not the whole
checkout flow.

Step 3 — LOGS: Filtering payment-service's logs to the 14:32 window
reveals the specific error:
  [ERROR] [payment-service] Connection timeout to payment-gateway-api
  after 5000ms — gateway host unreachable

This tells us WHY: the payment service can't reach an external
gateway, likely a network or third-party outage — a specific,
actionable finding that neither the metric (told us "something's
wrong") nor the trace (told us "it's the payment step") could have
revealed on their own.
```

## Production Considerations

- Push for structured, correlated logging (a shared request/trace ID present in logs, metrics, and traces alike) across services — without this correlation, moving from "the trace shows a problem in payment-service" to "here are payment-service's relevant log lines" requires manual, time-consuming cross-referencing instead of a direct lookup.
- SDETs don't need to build or own an organization's observability infrastructure (that's typically an SRE/platform engineering responsibility) — but knowing how to *use* existing dashboards, trace viewers, and log search tools effectively during investigation is a practical, expected skill.
- When a test fails against a real deployed environment (staging, a pre-prod environment) rather than a fully isolated local/CI setup, the application's own observability stack is often the fastest path to root cause — faster than trying to reproduce the issue purely through the test framework's own artifacts.

## Common Pitfalls

- Reaching for logs first when a metric or trace would answer the question faster — for "is this actually a widespread issue or a one-off," a metric dashboard answers in seconds what grepping through logs could take much longer to establish.
- Not knowing how to navigate to the application's own logs/traces/metrics when a test fails against a real environment, relying solely on the test framework's own (necessarily limited) view of what happened.
- Treating logs, metrics, and traces as redundant/interchangeable rather than complementary — each pillar is genuinely better suited to a different kind of question, as the example investigation demonstrates.
- Investigating a production-adjacent failure without correlating across pillars (jumping straight to reading logs without first identifying the right time window/service from a metric or trace), leading to a much slower, less targeted investigation.

## Interview Notes

- Be ready to explain the three pillars of observability and, critically, what question each is best suited to answer — not just definitions, but the practical "when do I reach for which one."
- Understand and be able to walk through a realistic investigation flow moving from a metric (something's wrong) → a trace (where) → logs (why) — this end-to-end narrative is what interviewers are really listening for.
- Be able to describe why correlation IDs (shared identifiers linking a trace to its corresponding logs) matter practically for investigation speed.

## References

- [OpenTelemetry — Observability Primer](https://opentelemetry.io/docs/concepts/observability-primer/)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)