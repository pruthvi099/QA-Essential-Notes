# Distributed Tracing Basics

## What It Is

Distributed tracing follows a single request's journey as it moves through multiple services in a microservices architecture, breaking it into **spans** — individual units of work (a database query, a call to another service, a computation) — assembled into a **trace**: the complete, hierarchical timeline of that one request's execution across every service it touched. This is the "where" pillar from [Observability Fundamentals](./observability-fundamentals.md), covered here in the depth needed to actually read and use a trace during investigation.

## Why It Matters

- In a microservices architecture, a single user-facing request (like submitting an order) can touch five, ten, or more internal services — without tracing, understanding which specific service in that chain is slow or failing requires painstakingly correlating logs across every service manually; a trace shows this directly, visually.
- This directly extends the [Levels of Testing](../00-start-here/levels-of-testing.md) and [Backend Verification Testing](../05-sql-database-testing/backend-verification-testing.md) concepts — a trace makes visible exactly which "layer" (in the sense of [API-to-Database Validation](../05-sql-database-testing/api-to-database-validation.md)) is responsible for an observed slowdown or failure, turning an abstract architectural diagram into concrete, per-request evidence.
- As microservices architectures have become the norm rather than the exception, tracing literacy is an increasingly practical, expected debugging skill for an SDET working on anything beyond a simple monolith.

## How It Works

**Trace structure:** a trace is a tree of spans — a **root span** representing the overall request, with **child spans** representing each sub-operation (a database call, a call to another service) nested underneath, each with its own start time, duration, and status (success/error).

**Reading a trace visually (as shown in tools like Jaeger, Datadog APM, or Honeycomb):** spans are typically displayed as a waterfall/Gantt-chart-style timeline — the width of each span bar shows its duration, and its horizontal position shows when it started relative to the overall request, making it immediately visually apparent which span is disproportionately slow or where an error occurred.

**Key concepts:**
- **Trace ID** — a unique identifier for the entire end-to-end request, the same correlation ID discussed in [Reading Application Logs for Debugging](./reading-application-logs-for-debugging.md), propagated across every service the request touches.
- **Span ID** — a unique identifier for one specific operation within the trace.
- **Parent-child relationships** — spans nest, showing which operation triggered which — e.g., "checkout" (root) → "validate-inventory" → "process-payment" → "charge-gateway" (each nested inside the one that called it).

## Example

**A textual representation of what a trace waterfall view shows** — the same structure a tool like Jaeger would render visually:

```text
Trace ID: 8f3a2b1c-...

checkout-request (root span)                    [========================] 4,200ms TOTAL
  ├─ validate-inventory                          [==] 180ms
  ├─ calculate-pricing                           [=] 45ms
  ├─ process-payment                             [====================] 3,850ms  ← clearly
  │                                                                       the dominant,
  │   ├─ charge-gateway (external API call)      [==================] 3,600ms ←  slow span
  │   └─ record-transaction (database write)     [==] 210ms
  └─ send-confirmation-email                     [=] 90ms

Visual reading: the checkout request took 4.2s total. Just from the
proportional WIDTH of each span bar, it's immediately obvious that
process-payment — specifically its charge-gateway sub-span — is
responsible for the vast majority of that time (3.6s of the 4.2s
total), not inventory validation, pricing, or email sending.

This directly answers "WHERE is the slowness," narrowing the
investigation from "checkout is slow" to "the external payment
gateway call specifically is slow" — precisely the kind of finding
that then sends the investigation to that service's LOGS (see
Reading Application Logs for Debugging) for the "WHY."
```

**Connecting trace investigation to a performance test finding**, tying this note back to [Identifying Performance Bottlenecks](../14-performance-testing/identifying-performance-bottlenecks.md):
```text
A k6 load test showed checkout's p95 latency degrading under load.
Pulling a trace for one of the slow requests during that test window
shows the SAME pattern as above — process-payment/charge-gateway
dominating the total time — confirming the bottleneck is an
EXTERNAL dependency (the payment gateway), not the application's
own code or database, directly informing where remediation effort
should focus.
```

## Production Considerations

- Trace propagation requires every service in the chain to correctly pass the trace context (trace ID, parent span ID) to the next service it calls — a service that doesn't participate in trace propagation creates a "gap" in the trace, making that service's internal behavior invisible to the tracing tool even though the overall request passed through it.
- Sampling matters at scale — tracing every single request in a high-traffic production system has real overhead cost, so many systems only trace a sampled percentage of requests; understanding your system's sampling rate matters when trying to find a trace for a *specific* failed request (it may not have been sampled/captured at all).
- Trace tools (Jaeger, Datadog APM, Honeycomb, Zipkin) each have somewhat different UIs but represent the same underlying waterfall/span concept — the reading skill transfers across tools even though the specific navigation doesn't.

## Common Pitfalls

- Not knowing how to find a trace for a *specific* failing request (versus browsing traces generally) — searching by trace ID (from a response header or log correlation ID) is the direct, targeted approach, much faster than browsing.
- Assuming a request was definitely traced and searching in vain for a trace that was never sampled/captured — worth confirming the system's sampling behavior before assuming a missing trace indicates a tooling problem.
- Misreading span nesting as sequential when it might actually represent parallel execution (some tracing UIs show parallel child spans as overlapping in the timeline, not sequential) — worth checking the specific tool's visual convention rather than assuming.
- Stopping the investigation at "this span is slow" without going further into that specific service's logs to find the actual root cause — a trace shows *where*, not *why*; it's the next investigative step (see [Reading Application Logs for Debugging](./reading-application-logs-for-debugging.md)), not the final answer.

## Interview Notes

- Be ready to explain trace/span structure precisely — trace ID for the whole request, span for each individual operation, parent-child nesting showing causality — and how this differs from a simple linear log sequence.
- Understand how to read a trace waterfall visually to identify the dominant, bottleneck span — a common, practical exercise given a described or shown trace.
- Be able to describe the full investigative chain: metric shows something's wrong → trace shows where → logs show why (connecting directly back to [Observability Fundamentals](./observability-fundamentals.md)) — this end-to-end fluency is what distinguishes real understanding from isolated fact recall.

## References

- [OpenTelemetry — Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Jaeger — Distributed Tracing](https://www.jaegertracing.io/docs/latest/)