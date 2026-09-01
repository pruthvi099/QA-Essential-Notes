# Identifying Performance Bottlenecks

## What It Is

This note covers reading performance test results (from k6 or JMeter, as covered in the previous two notes) to distinguish *where* a bottleneck actually lives — application code, database, or infrastructure/network — rather than stopping at "the system is slow under load." Correctly locating a bottleneck is what turns a performance test result into an actionable fix, connecting directly to [Backend Verification Testing](../05-sql-database-testing/backend-verification-testing.md) and [Transaction & Consistency Testing](../05-sql-database-testing/transaction-and-consistency-testing.md) for the database-layer investigation specifically.

## Why It Matters

- "The system is slow under load" is a starting observation, not a diagnosis — without identifying the actual bottleneck layer, a team can spend significant effort optimizing the wrong thing (scaling application servers when the real constraint is database connection pool exhaustion, for example).
- Different bottleneck layers have genuinely different fixes — an application-code bottleneck needs profiling and code optimization; a database bottleneck needs query/index optimization or connection pooling changes; an infrastructure bottleneck needs scaling or configuration changes — misdiagnosing which one you have wastes real engineering effort.
- This is where performance testing connects to genuine system-design understanding — being able to reason about *why* a system slows down under load, not just measure that it does, is a meaningfully more advanced and valuable skill.

## How It Works

**Signals pointing to an application-code bottleneck:**
- CPU utilization on application servers climbs toward 100% under load while database/infrastructure metrics stay healthy.
- Latency increases roughly linearly with load in a way that suggests single-threaded or inefficiently-parallelized processing.
- Profiling (application performance monitoring tools) shows time concentrated in specific application code paths, not waiting on external calls.

**Signals pointing to a database bottleneck:**
- Application server CPU stays low/healthy, but response times still degrade — the application is *waiting*, not *computing*.
- Database-specific metrics (query execution time, connection pool utilization, lock contention) show clear stress under the same load.
- A specific slow query identified via database query logs correlates directly with the endpoints showing the worst latency.

**Signals pointing to an infrastructure/network bottleneck:**
- Both application and database metrics look healthy, but end-to-end latency is still poor — often points to network bandwidth, load balancer configuration, or connection limits between layers.
- Errors specifically resembling connection timeouts/refusals (rather than application-level errors) under high concurrency.

## Example

**Correlating a k6 test result with backend metrics to distinguish an application versus database bottleneck — the actual investigative workflow, not just observing the symptom:**

```text
Observation: k6 load test shows p95 latency degrading badly at 500+
concurrent users (from 300ms at 100 users to 4.5s at 500 users).

Investigation step 1 — check application server resource metrics
  during the same test window:
  CPU utilization: 35% (NOT the bottleneck — plenty of headroom)
  Memory: stable, no signs of leak

Investigation step 2 — check database metrics during the same window:
  Connection pool utilization: 98% (near saturation)
  Average query time on the 'orders' table: climbing from 15ms
  to 800ms as load increases

Diagnosis: DATABASE bottleneck, specifically connection pool
exhaustion combined with a slow query — NOT an application-code
or infrastructure issue.

This directly determines the fix: increasing connection pool size
alone might help somewhat, but the growing query time under load
suggests a missing index or an inefficient query is the ROOT cause
— see Backend Verification Testing and SQL fundamentals for how to
investigate the specific query further.
```

**Using a database-specific query to identify the actual slow query correlating with the performance degradation** (connecting directly to [SQL Fundamentals for QA](../05-sql-database-testing/sql-fundamentals-for-qa.md)):
```sql
-- Identify slow queries during the load test window (PostgreSQL example)
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY mean_exec_time DESC
LIMIT 5;

-- Confirming the suspicion: a query missing an index, showing
-- mean_exec_time climbing disproportionately under concurrent load
```

**A clear decision framework for bottleneck triage, usable as a quick mental checklist during investigation:**
```text
1. Is application server CPU/memory maxed out?
   YES → likely application-code bottleneck, profile the code
   NO  → continue to step 2

2. Are database metrics (connection pool, query time, locks) showing
   stress correlating with the degradation?
   YES → likely database bottleneck, investigate specific slow queries
   NO  → continue to step 3

3. Are both app and DB healthy, but end-to-end latency still poor?
   → likely infrastructure/network bottleneck — check load balancer
     config, network bandwidth, inter-service connection limits
```

## Production Considerations

- Instrument application and database monitoring *before* running a load test, not after — trying to diagnose a bottleneck after the fact from k6/JMeter results alone, without correlated backend metrics, is significantly harder than having both data sources available from the start.
- Correlate performance test timestamps precisely with backend monitoring dashboards — the diagnostic value comes specifically from seeing which system's metrics degrade *at the same time* as the observed latency increase, not from looking at either data source in isolation.
- Fixing a misdiagnosed bottleneck (scaling application servers when the real issue is a database connection pool) can be genuinely costly — both in wasted engineering effort and in the false confidence of "we fixed it" when the real bottleneck remains.

## Common Pitfalls

- Stopping the investigation at "the system is slow under load" without correlating with backend metrics to identify which specific layer is actually constrained.
- Assuming the bottleneck is always the database (a common default assumption) without actually checking application server resource metrics first — sometimes the real bottleneck is genuinely in application code.
- Not having monitoring/observability instrumented during a load test, making bottleneck diagnosis after the fact much harder or impossible.
- Fixing the first plausible-looking issue found without confirming it's actually the *primary* constraint — a system under load can show multiple stressed metrics simultaneously, and identifying the actual limiting factor (versus a secondary symptom) requires care.

## Interview Notes

- Be ready to describe a systematic approach to bottleneck diagnosis — checking application metrics, then database metrics, then infrastructure — rather than guessing based on assumption.
- Understand the difference between application-code, database, and infrastructure bottleneck signals, and be able to give a concrete example of what each looks like in monitoring data.
- Be able to describe why correlating performance test results with backend monitoring (not just looking at the load test tool's own output) is essential for actual diagnosis, not just observation.

## References

- [k6 — Test Result Output](https://k6.io/docs/results-output/real-time/)
- [PostgreSQL — pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)