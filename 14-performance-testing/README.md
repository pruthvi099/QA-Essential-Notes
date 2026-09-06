# 14 — Performance Testing

Load, stress, spike, and soak testing — from fundamentals through k6/JMeter tooling, bottleneck diagnosis, CI integration, and overall strategy. Read [00-start-here](../00-start-here/) first for risk-based testing and entry/exit criteria, which this folder's baseline-setting builds on directly.

## Notes

1. [Performance Testing Fundamentals](./performance-testing-fundamentals.md) — Load/stress/spike/soak, and why percentiles beat averages
2. [Performance Testing with k6](./performance-testing-with-k6.md) — Code-first load testing with VUs, checks, and thresholds
3. [Performance Testing with JMeter](./performance-testing-with-jmeter.md) — Thread Groups, Samplers, and when JMeter is the practical choice
4. [Identifying Performance Bottlenecks](./identifying-performance-bottlenecks.md) — Distinguishing app, database, and infrastructure constraints
5. [Performance Testing in CI](./performance-testing-in-ci.md) — Tiered checks and trend tracking without blocking every PR
6. [Performance Testing Strategy & Baselines](./performance-testing-strategy-and-baselines.md) — Deriving baselines and risk-based test allocation

## Related

- [00 — Start Here](../00-start-here/) — risk-based testing and entry/exit criteria this folder's strategy builds on
- [05 — SQL & Database Testing](../05-sql-database-testing/) — database-layer investigation for bottleneck diagnosis
- [07 — CI/CD](../07-ci-cd/) — the tiering and trigger principles this folder's CI integration extends