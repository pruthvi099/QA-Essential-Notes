# Mobile App Performance Testing

## What It Is

Mobile app performance testing covers the metrics specific to mobile UX that don't have a direct web equivalent — app launch time (cold start vs. warm start), memory consumption, battery drain, and frame rate/scroll smoothness. This is distinct from backend/API performance testing (covered separately in [14-performance-testing](../14-performance-testing/)) — this note focuses on client-side, on-device performance characteristics an SDET should know how to measure and reason about.

## Why It Matters

- Mobile users are far less tolerant of slow launch times or janky scrolling than desktop web users, and mobile hardware constraints (limited RAM, battery) make performance issues both more common and more consequential — a memory leak that's a minor concern on a desktop can crash a mobile app entirely.
- Poor performance directly affects app store metrics (uninstall rates, ratings) in ways more immediate than most web performance issues — app store reviews frequently cite "slow" or "drains battery" as reasons for low ratings, giving performance testing very concrete business stakes.
- This is a genuinely differentiating skill area — many testers focus purely on functional correctness and never measure performance characteristics, so being able to speak to launch time, memory, and battery testing specifically signals broader, more complete mobile QA maturity.

## How It Works

**Key mobile performance metrics:**

1. **Cold start time** — time from tapping the app icon (app not in memory at all) to the first interactive screen. This is the most user-visible launch metric and the one most scrutinized by app store guidelines/reviews.
2. **Warm start time** — time to resume when the app is already partially in memory (backgrounded, not fully killed) — typically much faster than cold start, and a meaningfully different metric worth measuring separately.
3. **Memory usage** — how much RAM the app consumes, especially important on lower-end devices with limited memory where excessive usage can cause the OS to kill the app entirely.
4. **Battery consumption** — background activity, location polling, and network chattiness are common battery drain culprits worth specifically profiling.
5. **Frame rate / scroll jank** — smoothness of animations and scrolling, typically measured in frames per second (60fps is the common smooth-scrolling target on most modern devices).

**Tools:**
- **Android**: Android Studio Profiler (CPU, memory, network, energy), `adb shell am start -W` for launch time measurement.
- **iOS**: Xcode Instruments (Time Profiler, Allocations, Energy Log).
- Both platforms provide command-line and GUI tooling suitable for both manual investigation and scripted, repeatable measurement in CI.

## Example

**Measuring Android cold start time via ADB, in a scriptable way suitable for CI tracking over time:**
```bash
# Force-stop the app first to guarantee a genuine cold start
adb shell am force-stop com.example.shoppingapp

# Launch and measure — outputs TotalTime (ms) among other metrics
adb shell am start -W -n com.example.shoppingapp/.MainActivity
```

```python
import subprocess
import re

def measure_cold_start_time(package_name: str, activity: str) -> int:
    subprocess.run(["adb", "shell", "am", "force-stop", package_name])

    result = subprocess.run(
        ["adb", "shell", "am", "start", "-W", "-n", f"{package_name}/{activity}"],
        capture_output=True, text=True
    )

    # Parse the TotalTime line from adb's output
    match = re.search(r"TotalTime:\s*(\d+)", result.stdout)
    return int(match.group(1)) if match else None

def test_cold_start_time_within_acceptable_threshold():
    launch_time_ms = measure_cold_start_time(
        "com.example.shoppingapp", ".MainActivity"
    )

    # A sanity threshold, not a brittle exact value — catches real
    # regressions (e.g., a launch time doubling) without failing on
    # normal minor variance between runs
    assert launch_time_ms is not None
    assert launch_time_ms < 3000, f"Cold start took {launch_time_ms}ms, exceeding 3000ms threshold"
```

**Tracking performance metrics over time as a CI trend, rather than a pass/fail gate alone** — since a single run's timing can vary, trend tracking often reveals gradual regressions a strict threshold alone would miss:

```python
def test_and_log_cold_start_trend(db_connection):
    launch_time_ms = measure_cold_start_time("com.example.shoppingapp", ".MainActivity")

    cursor = db_connection.cursor()
    cursor.execute(
        "INSERT INTO performance_metrics (metric, value, recorded_at) VALUES (%s, %s, NOW())",
        ("cold_start_time_ms", launch_time_ms)
    )
    db_connection.commit()

    # Compare against a rolling average of recent builds, not just a
    # fixed threshold — catches gradual creep even if no single build
    # crosses an absolute limit
    cursor.execute("""
        SELECT AVG(value) FROM performance_metrics
        WHERE metric = 'cold_start_time_ms'
        AND recorded_at > NOW() - INTERVAL '30 days'
    """)
    rolling_avg = cursor.fetchone()[0]

    assert launch_time_ms < rolling_avg * 1.5, (
        f"Cold start time ({launch_time_ms}ms) is significantly worse than "
        f"the 30-day rolling average ({rolling_avg}ms) — possible regression"
    )
```

## Production Considerations

- Measure performance metrics on representative low-end devices, not just the newest flagship models — a launch time or memory profile that's fine on a high-end device can be a genuine problem on the lower-end devices a meaningful portion of real users own, especially in markets with more budget-device usage.
- Track performance trends over time (as shown above) rather than relying solely on a single absolute threshold — this catches gradual regressions (each release adding a small amount of overhead) that no single release would individually trigger a hard failure for.
- Performance testing tooling (Profiler, Instruments) is often more suited to manual, investigative deep-dives than fully automated CI gating — a practical approach combines lightweight, scriptable metrics (like launch time via ADB) in CI with periodic, deeper manual profiling sessions for investigating specific concerns.

## Common Pitfalls

- Testing performance exclusively on high-end devices/emulators, missing real-world performance issues that only manifest on memory-constrained, lower-end hardware.
- Using a single hard threshold without accounting for normal run-to-run variance, causing either flaky test failures (threshold too tight) or missed regressions (threshold too loose to actually catch meaningful degradation).
- Not distinguishing cold start from warm start when reporting or investigating a launch time concern — conflating the two produces confusing, inconsistent measurements depending on the app's actual memory state at test time.
- Treating performance testing as purely a manual, occasional activity rather than incorporating at least basic scriptable metrics (like the ADB launch time example) into regular CI runs where feasible.

## Interview Notes

- Be ready to name the mobile-specific performance metrics (cold/warm start, memory, battery, frame rate) that don't have a direct web-testing equivalent — a foundational, commonly asked distinction.
- Understand why measuring performance only on high-end devices gives a misleading picture, and be able to explain the real-world stakes (app store ratings, uninstalls) tied to mobile performance specifically.
- Be able to describe how you'd track a performance metric over time (trend-based) rather than relying solely on a single pass/fail threshold — this shows more mature, production-oriented performance testing thinking.

## References

- [Android Developers — App Startup Time](https://developer.android.com/topic/performance/vitals/launch-time)
- [Apple Developer — Instruments Help](https://help.apple.com/instruments/mac/current/)