# Reading Application Logs for Debugging

## What It Is

This note covers reading and querying an application's own production/staging logs during test failure investigation — structured (JSON) logging conventions, log levels, and effective log-search queries — extending [Observability Fundamentals](./observability-fundamentals.md)'s "why" pillar and [Logging Strategy for Test Frameworks](../10-test-framework-design/logging-strategy-for-test-frameworks.md)'s framework-logging principles to the application side.

## Why It Matters

- A test failure against a real environment often can't be fully diagnosed from the test's own artifacts (trace, screenshot) alone — the application's own logs frequently contain the actual root cause (an unhandled exception, a downstream service timeout) that never surfaces in the browser/API response the test observed.
- Structured (JSON) logs are searchable and filterable in ways unstructured plain-text logs aren't — knowing how to construct an effective query (filtering by request ID, service, time window, severity) is a practical, everyday investigative skill, not a theoretical one.
- This is a common gap for SDETs who are fluent in test-framework-level debugging (see [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md)) but haven't developed comfort navigating the application's own log aggregation platform (Datadog, Splunk, ELK/OpenSearch, CloudWatch).

## How It Works

**Structured logging** — each log entry is a JSON object with consistent fields (timestamp, level, service, message, and contextual fields like `request_id`, `user_id`), rather than a free-form text line — this is what makes logs queryable at scale rather than requiring `grep`-style text search across raw files.

**Log levels, and what to search for at each during investigation:**
- **ERROR** — the first, most direct place to look when investigating a failure — represents something that actually went wrong.
- **WARN** — worth checking for near the failure's timestamp — often reveals a degraded-but-not-yet-failed precursor condition (a retry, a fallback path taken).
- **INFO** — useful for reconstructing the sequence of events leading up to a failure, once ERROR/WARN entries have narrowed the relevant time window.
- **DEBUG** — typically not enabled by default in production; may need to be temporarily enabled for a specific investigation if ERROR/WARN/INFO don't provide enough detail.

**The core investigative technique: correlation ID lookup.** If a test failure has an identifiable request ID (from a response header, or a trace — see [Distributed Tracing Basics](./distributed-tracing-basics.md)), searching logs for that exact ID across all services immediately surfaces every log line related to that specific failing request, cutting through unrelated concurrent traffic entirely.

## Example

**A realistic log-search investigation, showing the query refinement process:**

```text
Starting point: An E2E test failed with "checkout submission returned
500." The API response included a header: X-Request-ID: req-8f3a2b1c

Step 1 — Search logs for the exact request ID across ALL services
(the single most targeted, effective first query):

  Query: request_id:"req-8f3a2b1c"

  Result: Immediately surfaces every log line related to THIS
  specific failing request, across order-service, payment-service,
  and inventory-service — no need to guess which service or sift
  through unrelated concurrent traffic.

Step 2 — Among the returned lines, the ERROR-level entry is the
most direct lead:

  {
    "timestamp": "2026-09-15T14:32:07Z",
    "level": "ERROR",
    "service": "payment-service",
    "request_id": "req-8f3a2b1c",
    "message": "Failed to process payment: gateway timeout",
    "gateway": "stripe",
    "timeout_ms": 5000
  }

Step 3 — Checking for a WARN entry immediately preceding it reveals
a precursor:

  {
    "timestamp": "2026-09-15T14:32:01Z",
    "level": "WARN",
    "service": "payment-service",
    "request_id": "req-8f3a2b1c",
    "message": "Retrying gateway connection (attempt 2/3)"
  }

Full picture: the payment gateway was slow/unreachable, the service
retried per its configured policy, and ultimately timed out after
exhausting retries — a clear, actionable root cause found via a
single, precisely-targeted correlation ID search.
```

**A broader search when no correlation ID is available, narrowing by time window and service instead:**
```text
Query: service:"payment-service" AND level:"ERROR" AND @timestamp:[2026-09-15T14:30:00Z TO 2026-09-15T14:35:00Z]

# Less targeted than a correlation ID lookup, but still far more
# effective than searching raw, unstructured logs without any
# service/time/level filtering at all
```

## Production Considerations

- Advocate for (or verify the existence of) a correlation/request ID that's consistently propagated across every service a request touches — this single practice is what makes the difference between a fast, targeted investigation and a slow, manual cross-referencing exercise across unrelated concurrent logs.
- Learn your organization's specific log aggregation platform's query syntax (Datadog, Splunk, ELK/OpenSearch, CloudWatch Logs Insights each have their own query language) — the underlying investigative principles transfer, but the exact syntax doesn't, so hands-on familiarity with the actual tool in use matters.
- When DEBUG-level detail is needed but not normally enabled, know the process for temporarily enabling it for a specific investigation (and remembering to disable it afterward) — DEBUG logging left on indefinitely in production has real performance and storage cost implications.

## Common Pitfalls

- Searching logs without any correlation ID, service, or time-window filter, sifting through an overwhelming, unfiltered stream of unrelated concurrent traffic instead of a targeted query.
- Not knowing the response header or trace field that carries the correlation ID for a given application, missing the single most effective investigative shortcut available.
- Stopping at the first ERROR entry found without checking for a WARN precursor nearby in time — the full causal picture (as in the example, a retry-then-timeout sequence) is often only visible by looking slightly before the error itself.
- Assuming DEBUG-level detail is always available — many production systems deliberately don't log at DEBUG level by default, and assuming it exists wastes time searching for detail that was never captured.

## Interview Notes

- Be ready to describe how you'd investigate a test failure using an application's logs, specifically mentioning correlation/request ID lookup as the most effective starting technique — this is a strong, specific, practical answer.
- Understand the log level hierarchy and what each level is useful for during investigation, not just as an abstract severity scale.
- Be able to name at least one log aggregation platform (Datadog, Splunk, ELK, CloudWatch) and describe, at a high level, how you'd construct a targeted search query in one.

## References

- [OpenTelemetry — Logs](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Elastic — What is Structured Logging?](https://www.elastic.co/what-is/structured-logging)