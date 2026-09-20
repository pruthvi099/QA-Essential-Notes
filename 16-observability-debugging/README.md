# 16 — Observability & Debugging

Reading logs, traces, and metrics to debug beyond what test frameworks alone reveal — plus browser DevTools, unreproducible production issues, and monitoring/alerting basics. Read [02-automation-python-playwright](../02-automation-python-playwright/) first for the test-framework-level debugging this folder extends into the application's own observability stack.

## Notes

1. [Observability Fundamentals](./observability-fundamentals.md) — Logs, metrics, and traces — the three pillars and when to use each
2. [Reading Application Logs for Debugging](./reading-application-logs-for-debugging.md) — Structured logging, log levels, and correlation ID search
3. [Distributed Tracing Basics](./distributed-tracing-basics.md) — Spans, waterfalls, and finding the bottleneck service
4. [Using Browser DevTools for Debugging](./using-browser-devtools-for-debugging.md) — Network, Console, and Storage panels for client-side triage
5. [Debugging Flaky Production Issues](./debugging-flaky-production-issues.md) — Investigating unreproducible bugs from historical evidence
6. [Monitoring & Alerting Basics for QA](./monitoring-and-alerting-basics-for-qa.md) — Reading dashboards and contributing to alert calibration

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — test-framework-level debugging this folder extends
- [14 — Performance Testing](../14-performance-testing/) — the metrics and bottleneck concepts shared with this folder
- [07 — CI/CD](../07-ci-cd/) — where CI failures connect to the application's own observability stack