# Multi-Region & Scalability Testing

## What It Is

This note covers testing behavior specific to multi-region cloud deployments (data residency, regional failover, cross-region latency) and auto-scaling infrastructure (testing the *transition* behavior as infrastructure scales up/down, not just steady-state performance) — the two remaining cloud-native infrastructure behaviors introduced conceptually in [Cloud Testing Fundamentals](./cloud-testing-fundamentals.md), covered here in testing depth.

## Why It Matters

- Multi-region architecture introduces genuine correctness concerns beyond performance — data residency requirements (some data must legally stay in a specific region), and regional failover correctness (does the application actually recover gracefully if one region becomes unavailable) — these are testable, real risks, not just performance nice-to-haves.
- Auto-scaling's *transition* period — the gap between a traffic spike starting and new capacity actually coming online — is a distinct, testable failure window that steady-state load testing (see [Performance Testing Fundamentals](../14-performance-testing/performance-testing-fundamentals.md)) doesn't directly cover, since a steady-state test assumes capacity is already provisioned.
- This is genuinely advanced, infrastructure-adjacent testing territory — understanding it shows an SDET thinking about correctness and resilience at a systems-architecture level, not just application-feature level.

## How It Works

**Multi-region testing concerns:**
1. **Data residency compliance** — verifying data that must legally stay in a specific region (e.g., EU user data under GDPR-adjacent requirements) actually does, and isn't inadvertently replicated or processed in a non-compliant region.
2. **Regional failover** — if a primary region becomes unavailable, does traffic correctly route to a healthy secondary region, and does the application behave correctly during that transition (not just after it completes)?
3. **Cross-region latency** — for applications with components split across regions, does inter-region communication latency stay within acceptable bounds, and does the application handle a temporary cross-region connectivity issue gracefully?

**Auto-scaling transition testing:**
- **Scale-up lag** — during the gap between a traffic spike starting and new instances actually becoming available and healthy, does the existing capacity degrade gracefully (slower, but still correct) or fail outright (errors, dropped requests)? This directly extends the spike-testing concept from [Performance Testing Fundamentals](../14-performance-testing/performance-testing-fundamentals.md), specifically examining the *scaling* behavior during that spike, not just the final steady-state result.
- **Scale-down correctness** — when auto-scaling removes capacity as traffic drops, are in-flight requests on the instances being terminated handled gracefully (drained/completed) rather than abruptly dropped?

## Example

**Testing regional failover behavior — simulating a primary region outage and verifying correct failover:**
```python
import boto3

def test_application_fails_over_to_secondary_region():
    route53 = boto3.client('route53')

    # Simulate primary region becoming unhealthy by manipulating
    # the health check status in a controlled test environment
    # (this would use the actual health-check mechanism your
    # architecture relies on — Route53 health checks, a load
    # balancer's target health, etc.)
    simulate_region_health_check_failure(region='us-east-1')

    # Wait for DNS/routing to detect and react to the failure
    time.sleep(30)

    # Verify traffic now correctly routes to the secondary region
    response = requests.get('https://api.example.com/health')
    assert response.headers.get('X-Served-From-Region') == 'us-west-2'

    # Verify the application is FUNCTIONALLY correct in the
    # secondary region, not just reachable — a failed-over region
    # that's reachable but missing recent data would be a much
    # subtler, more dangerous failure than simple unreachability
    order_response = requests.get('https://api.example.com/api/orders/501')
    assert order_response.status_code == 200
```

**Testing auto-scaling transition behavior — observing what happens DURING a scale-up event, not just before/after:**
```python
def test_error_rate_during_autoscale_transition():
    # Trigger a sharp traffic spike (similar to Performance Testing
    # Fundamentals' spike test pattern), while specifically monitoring
    # error rate THROUGHOUT the scaling transition, not just at the end
    results = run_spike_load_test(
        baseline_rps=50,
        spike_rps=2000,
        spike_duration_seconds=180,
    )

    # Break down results into time windows to specifically examine
    # the SCALING TRANSITION period, not just the final steady state
    transition_window = results.filter_by_time(start=0, end=60)  # first 60s of the spike
    steady_state_window = results.filter_by_time(start=120, end=180)  # after scaling caught up

    # Some elevated error rate during the TRANSITION may be an
    # accepted, understood trade-off — but it should be BOUNDED
    # and should IMPROVE once scaling catches up, not remain elevated
    assert transition_window.error_rate < 0.05   # some tolerance during transition
    assert steady_state_window.error_rate < 0.005  # should be back to normal once scaled
```

## Production Considerations

- Multi-region failover testing should be run periodically (not just once, at initial setup) against real infrastructure — failover mechanisms that are never actually exercised are a common, real source of "we thought failover worked" surprises during an actual incident (this is the same principle behind disaster recovery drills more broadly).
- Auto-scaling transition testing requires the load test tooling to report time-windowed results (not just an aggregate summary), specifically to isolate the transition period's behavior from steady-state behavior — see [Performance Testing with k6](../14-performance-testing/performance-testing-with-k6.md) for tooling capable of this granularity.
- Data residency testing often has real legal/compliance stakes — coordinate with legal/compliance stakeholders on what specifically needs verification, similar to the [WCAG Conformance Levels](../15-accessibility-visual-testing/wcag-conformance-levels.md) pattern of confirming exact regulatory requirements rather than assuming a default.

## Common Pitfalls

- Testing multi-region failover only once, at initial setup, and never again — failover mechanisms can silently degrade or break as the application evolves, and periodic re-testing (or automated failover drills) is what actually confirms they still work.
- Testing auto-scaling only via steady-state load tests, missing the specific, bounded-but-real error rate increase that can occur during the scaling transition itself.
- Assuming data residency compliance without specific verification — a multi-region architecture can inadvertently replicate or process data in a non-compliant region through a misconfigured backup, cache, or logging pipeline that wasn't originally considered.
- Not distinguishing "the failed-over region is reachable" from "the failed-over region is functionally correct" — a region that responds but is missing recent replicated data is a subtler, more dangerous failure mode than simple unreachability.

## Interview Notes

- Be ready to describe how you'd test regional failover — simulating an outage, verifying both reachability AND functional correctness in the secondary region — a common, practical distinction interviewers probe for specifically.
- Understand why auto-scaling transition behavior needs separate, time-windowed testing distinct from steady-state load testing, and be able to describe what "acceptable" transition behavior looks like (bounded, temporary degradation, not sustained failure).
- Be able to explain data residency as a genuine, testable compliance concern in multi-region architectures, not just a performance/availability consideration.

## References

- [AWS — Disaster Recovery of Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Google Cloud — Multi-Region Deployment Architecture](https://cloud.google.com/architecture/deploy-multi-region-google-kubernetes-engine)