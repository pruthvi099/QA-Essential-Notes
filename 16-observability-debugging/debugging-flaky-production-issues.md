# Debugging Flaky Production Issues

## What It Is

This note covers investigating intermittent bugs reported in production that can't be reliably reproduced on demand — a genuinely different, harder investigative problem than the test-suite flakiness covered in [Flaky Test Handling](../02-automation-python-playwright/flaky-test-handling.md), since here the "test" is a real user's unrepeatable experience, and the investigation relies entirely on the observability tools from earlier notes in this folder rather than on re-running a script.

## Why It Matters

- Unlike a flaky automated test (which can usually be re-run, traced, and eventually reproduced deterministically), an intermittent production issue reported by a single user often has no reliable reproduction steps at all — the entire investigation depends on reconstructing what happened after the fact from logs, traces, and metrics.
- This is where the observability skills from this whole folder — logs, traces, metrics — genuinely come together in their highest-value, most realistic application: piecing together a story from incomplete, historical evidence rather than watching a live failure happen.
- This is a distinctly senior-level investigative skill, and being able to describe a structured approach to it (rather than "I'd just try to reproduce it") is a strong signal in interviews for more experienced SDET/production-support-adjacent roles.

## How It Works

**A structured approach to investigating an unreproducible production issue:**

1. **Gather everything available from the report itself** — exact timestamp (or as close as the reporter can provide), the specific user/account affected if known, exact error message or symptom described, browser/device if relevant.
2. **Narrow the time window using metrics first** — check whether there was a broader anomaly (error rate spike, latency increase) around the reported time, which both confirms the report is plausible and narrows the investigation window (per [Observability Fundamentals](./observability-fundamentals.md)'s metric → trace → logs flow).
3. **Search for the specific request via correlation ID if available**, or by narrowing logs to the affected user/account and time window if not (per [Reading Application Logs for Debugging](./reading-application-logs-for-debugging.md)).
4. **Look for a pattern across multiple occurrences**, if the issue has been reported more than once — a single occurrence might be a fluke; a recurring pattern (always at a specific time, always for a specific user segment, always after a specific action) is much more diagnostically useful.
5. **Form and test a hypothesis** — once a plausible cause emerges from the log/trace evidence, verify it independently (does the timing/pattern match other known factors, like a deploy, a traffic spike, a third-party outage) before concluding.

## Example

A realistic walkthrough applying this structured approach to a vague, hard-to-reproduce report:

```text
Report: "A customer says their order total was wrong yesterday
afternoon, but it looked fine when I checked it just now. They
don't remember the exact time."

Step 1 — Gather what's available:
  User account: known (from the support ticket)
  Approximate time: "yesterday afternoon" — vague, needs narrowing
  Symptom: order total was wrong, specific order ID known

Step 2 — Check metrics for that day, since we have a rough afternoon
window and a suspicion this might not be isolated to one user:
  Reviewing the "order calculation error" metric shows a small but
  real spike between 14:00-14:15 the previous day — this both
  narrows the time window significantly AND suggests this wasn't
  an isolated, single-user fluke.

Step 3 — Search logs for that specific order ID, now with a much
narrower time window to search within:
  {
    "timestamp": "2026-09-14T14:07:23Z",
    "level": "WARN",
    "service": "pricing-service",
    "message": "Discount cache miss — falling back to stale value",
    "order_id": "ord-7731"
  }

Step 4 — Check for a pattern: were OTHER orders in that same 14:00-
14:15 window also affected?
  Query: service:"pricing-service" AND message:"discount cache miss"
  AND @timestamp:[2026-09-14T14:00:00Z TO 2026-09-14T14:15:00Z]

  Result: 23 other orders show the same warning in that window —
  confirming this wasn't a one-off fluke but a real, bounded incident
  affecting a specific set of orders during a specific 15-minute window.

Step 5 — Hypothesis and verification:
  Hypothesis: a discount pricing cache had a brief outage/staleness
  issue during that window, causing a temporary fallback to
  potentially outdated discount values.
  Verification: checking the cache infrastructure's own metrics
  confirms a brief connectivity blip to the cache layer at 14:03-14:12,
  matching the affected order window precisely.

Conclusion: root cause identified with confidence, despite having
NO way to directly reproduce the original customer's experience —
the investigation was built entirely from historical evidence
across metrics, logs, and a pattern search, not from reproduction.
```

## Production Considerations

- Even a vague report ("sometime yesterday afternoon") is often narrowable using metrics first, before diving into detailed log search — this ordering (broad signal → narrow window → specific logs) saves significant time compared to searching logs across a wide, unfocused time range.
- Checking whether an issue affected multiple users/orders (not just the one reported) is valuable both for scoping the actual impact (are there other affected customers who haven't reported it?) and for confirming the finding isn't coincidental.
- Document the investigation's evidence trail clearly when writing up a root-cause finding — since there's no reliable reproduction to point to, the log/trace/metric evidence chain IS the proof, and it should be preserved and referenced clearly for anyone reviewing the finding later.

## Common Pitfalls

- Giving up on an investigation because the issue "can't be reproduced," when a structured, evidence-based approach using existing logs/metrics/traces can often still identify a confident root cause without ever needing to reproduce it live.
- Searching logs across an unnecessarily wide time window because the initial report was vague, instead of first using metrics to narrow the window (as shown in the example) before diving into detailed log search.
- Concluding a root cause from a single data point without checking for a broader pattern — a single log line can be misleading or coincidental, while a pattern across many similar occurrences is much stronger evidence.
- Not preserving/documenting the evidence trail once a root cause is found, making it hard for anyone else to verify or reference the finding later, especially since there's no reproducible test case to fall back on.

## Interview Notes

- Be ready to describe a structured approach to investigating an unreproducible production issue — gather report details, narrow with metrics, search targeted logs, look for a pattern, form and verify a hypothesis — rather than "I'd just try to reproduce it," which doesn't work for genuinely intermittent issues.
- Understand why checking for a broader pattern (other affected users/orders) matters both diagnostically and for scoping real business impact.
- Be able to describe a realistic example (like the one above) showing how logs, metrics, and pattern-matching combine to reach a confident conclusion without ever reproducing the original symptom directly — this end-to-end narrative is the strongest possible answer to this topic.

## References

- [Google SRE Book — Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)