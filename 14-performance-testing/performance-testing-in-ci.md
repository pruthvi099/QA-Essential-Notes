# Performance Testing in CI

## What It Is

This note covers integrating performance checks into CI pipelines without the cost/time trade-offs blocking every PR — applying the staging principles from [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md) and [Smoke vs. Regression in CI](../07-ci-cd/smoke-vs-regression-in-ci.md) specifically to performance testing, which has a genuinely different cost profile (slower, resource-intensive) than typical functional test tiers.

## Why It Matters

- Full-scale load tests (hundreds of virtual users, several minutes duration) are far too slow and resource-intensive to run on every PR — but performance regressions are exactly the kind of issue that's much cheaper to catch early than after they've shipped, creating a real tension worth resolving deliberately.
- This mirrors the same trigger-tiering decision from [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md), applied to a test category with its own distinctive cost profile — the "what runs on PR vs. scheduled" question needs its own specific answer for performance tests, not just an inherited default.
- Performance regression detection specifically benefits from trend tracking over time (similar to [Mobile App Performance Testing](../06-mobile-testing/mobile-app-performance-testing.md)'s launch-time trend approach) rather than only a single pass/fail threshold, since gradual performance degradation across many small changes is a real, common pattern.

## How It Works

**A tiered approach to performance testing in CI:**
1. **Lightweight smoke-level performance check on PR** — a small, fast load test (few virtual users, short duration) checking for gross regressions only — not comprehensive, but fast enough to run on every PR without meaningfully slowing feedback.
2. **Full load/stress/soak tests on a schedule** — the comprehensive tests from [Performance Testing Fundamentals](./performance-testing-fundamentals.md), running nightly or pre-release, mirroring the full-regression-suite scheduling pattern from [Smoke vs. Regression in CI](../07-ci-cd/smoke-vs-regression-in-ci.md).
3. **Trend tracking** — recording key metrics (p95 latency, throughput) from every scheduled run over time, catching gradual degradation that no single run's threshold would individually trigger.

## Example

**A lightweight PR-level performance smoke check — fast enough to run on every PR:**
```yaml
# .github/workflows/perf-smoke.yml
name: Performance Smoke Check
on:
  pull_request:
    branches: [main]

jobs:
  perf_smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run lightweight k6 smoke check
        run: |
          docker run -i grafana/k6 run - <<EOF
          import http from 'k6/http';
          import { check } from 'k6';

          export const options = {
            vus: 10,              // small VU count — just enough to catch gross regressions
            duration: '30s',       // short — fast enough for PR feedback
            thresholds: {
              http_req_duration: ['p(95)<1000'],   // a LOOSE threshold, catching only gross regressions
            },
          };

          export default function () {
            const res = http.get('https://pr-preview.example.com/api/products');
            check(res, { 'status is 200': (r) => r.status === 200 });
          }
          EOF
```

**The full, comprehensive performance suite, running on a schedule instead — mirroring the nightly regression pattern:**
```yaml
# .github/workflows/perf-full.yml
name: Full Performance Test Suite
on:
  schedule:
    - cron: '0 3 * * *'   # nightly, not PR-blocking — full tests take too long

jobs:
  full_perf_suite:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run full load test
        run: k6 run checkout-load-test.js --out json=results.json

      - name: Store results for trend tracking
        run: |
          python scripts/extract_metrics.py results.json >> perf-history.csv
          # Appends this run's p95/throughput to a tracked history,
          # enabling the trend-based comparison shown below

      - name: Compare against rolling baseline
        run: python scripts/check_perf_trend.py perf-history.csv
        # Fails the build if THIS run's p95 is significantly worse
        # than the recent rolling average — catching gradual
        # degradation a single absolute threshold might miss
```

**A trend-comparison script's core logic, extending the same rolling-average pattern from [Mobile App Performance Testing](../06-mobile-testing/mobile-app-performance-testing.md):**
```python
import pandas as pd

def check_performance_trend(history_csv: str):
    history = pd.read_csv(history_csv)
    recent_avg_p95 = history['p95_latency'].tail(10).mean()
    latest_p95 = history['p95_latency'].iloc[-1]

    if latest_p95 > recent_avg_p95 * 1.3:
        raise SystemExit(
            f"Performance regression detected: latest p95 ({latest_p95}ms) "
            f"is 30%+ worse than the 10-run rolling average ({recent_avg_p95}ms)"
        )
```

## Production Considerations

- Keep the PR-level performance smoke check genuinely lightweight and its threshold deliberately loose — its purpose is catching gross, obvious regressions fast, not comprehensive performance validation; trying to make it thorough defeats the entire reason for tiering (fast PR feedback).
- Trend tracking requires persisting results across runs (a database, or even a simple committed CSV/artifact) — without historical data, every run is evaluated in isolation and gradual degradation goes undetected, the same insight from [Mobile App Performance Testing](../06-mobile-testing/mobile-app-performance-testing.md) applied here.
- Performance test environments need to be reasonably consistent run-to-run (similar resource allocation, not sharing capacity with unrelated concurrent CI jobs) — inconsistent test environment conditions introduce noise that makes trend comparison unreliable.

## Common Pitfalls

- Running full-scale load tests on every PR, creating an unacceptably slow feedback loop — mirroring the exact anti-pattern warned against in [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md), now specifically for performance tests' particularly high cost.
- Relying only on a single absolute threshold without trend tracking, missing gradual performance degradation that accumulates across many individually-small regressions.
- Setting PR-level smoke check thresholds too strict, causing frequent false-positive failures on a check meant to be fast and lightweight — undermining trust in the check the same way an overly strict quality gate erodes trust generally (see [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md)).
- Running performance tests on inconsistent, shared, or noisy CI infrastructure, introducing enough run-to-run variance that trend comparisons become unreliable.

## Interview Notes

- Be ready to describe a tiered performance testing strategy for CI — lightweight PR-level checks plus comprehensive scheduled runs — mirroring and applying the smoke/regression tiering principle from [Smoke vs. Regression in CI](../07-ci-cd/smoke-vs-regression-in-ci.md) to this specific, higher-cost test category.
- Understand why trend tracking (not just a single threshold) matters specifically for catching gradual performance regressions across many small changes.
- Be able to explain the cost/benefit trade-off that makes full performance testing unsuitable for every-PR execution, connecting back to general CI trigger design principles.

## References

- [k6 — CI/CD Integration](https://k6.io/docs/testing-guides/automated-performance-testing/)
- [Grafana — k6 GitHub Action](https://github.com/grafana/k6-action)