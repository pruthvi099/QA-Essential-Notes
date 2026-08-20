# Mobile Network Condition Testing

## What It Is

Mobile network condition testing verifies an app's behavior under realistic, imperfect network conditions — offline, slow (2G/3G-equivalent), intermittent connectivity, and the transition between states (going offline mid-request, reconnecting) — rather than only testing against a fast, stable network like most local development and CI environments default to.

## Why It Matters

- Mobile users experience unreliable connectivity far more often than desktop users — subway commutes, rural areas, weak WiFi — making network resilience testing disproportionately important for mobile apps specifically, more so than for most web applications.
- Apps that assume network calls always succeed quickly tend to fail ungracefully under real-world conditions — hanging indefinitely, crashing, or silently losing user-entered data (a form submission that fails invisibly when connectivity drops mid-request).
- This connects directly to [Mocking & Network Interception](../03-typescript-playwright/mocking-and-network-interception.md)'s web-based approach, applied here at the OS/network layer specific to mobile testing tools — the same underlying testing philosophy (simulate the edge case, verify graceful handling), different mechanism.

## How It Works

**Common network conditions to test:**
1. **Fully offline** — no connectivity at all; verify the app shows a clear offline state rather than hanging or crashing.
2. **Slow network** (throttled to 2G/3G-equivalent speeds) — verify loading states, timeouts, and perceived performance remain acceptable.
3. **Intermittent connectivity** — connection drops and reconnects during use; verify in-flight requests are retried or fail gracefully, not silently lost.
4. **Transition scenarios** — going offline mid-request (does a form submission that was in-flight get lost, retried, or clearly reported as failed to the user?), or coming back online (does the app auto-recover/sync, or require a manual refresh?).

**Testing approaches:**
- **Device/emulator network throttling** — most emulators and some real-device testing tools provide built-in network condition simulation (throttled speed, packet loss, offline toggle).
- **Airplane mode toggling** — a straightforward, realistic way to simulate a hard offline transition, especially valuable for testing the mid-request drop scenario.
- **Device farm network simulation** — many cloud device farms (see [Mobile Device Farms & Cloud Testing](./mobile-device-farms-and-cloud-testing.md)) offer network condition profiles as part of their test configuration.

## Example

**Python — testing offline behavior using Appium's network connection control:**
```python
def test_app_shows_offline_state_when_disconnected(driver):
    # Disable both WiFi and mobile data (Android-specific network control)
    driver.set_network_connection(0)  # 0 = airplane mode / fully offline

    driver.find_element("accessibility id", "refresh_feed_button").click()

    offline_banner = driver.find_element("accessibility id", "offline_banner")
    assert offline_banner.is_displayed()
    assert "no internet connection" in offline_banner.text.lower()

    # App should NOT show a raw error or hang indefinitely
    error_screen = driver.find_elements("accessibility id", "generic_error_screen")
    assert len(error_screen) == 0

def test_app_recovers_when_connection_restored(driver):
    driver.set_network_connection(0)  # go offline
    driver.find_element("accessibility id", "refresh_feed_button").click()
    assert driver.find_element("accessibility id", "offline_banner").is_displayed()

    driver.set_network_connection(6)  # restore WiFi + mobile data

    # App should auto-recover without requiring a manual app restart
    import time
    time.sleep(2)
    feed_content = driver.find_element("accessibility id", "feed_list")
    assert feed_content.is_displayed()

def test_form_submission_lost_connection_mid_request(driver):
    driver.find_element("accessibility id", "review_text_input").send_keys(
        "Great product, highly recommend!"
    )

    # Go offline immediately as the submission is triggered, simulating
    # a real-world drop mid-request rather than a clean offline start
    driver.find_element("accessibility id", "submit_review_button").click()
    driver.set_network_connection(0)

    # The app should clearly inform the user the submission failed,
    # rather than silently losing the entered review text
    error_message = driver.find_element("accessibility id", "submission_failed_message")
    assert error_message.is_displayed()

    # And the user's typed content should NOT be lost, so they can retry
    review_input = driver.find_element("accessibility id", "review_text_input")
    assert review_input.text == "Great product, highly recommend!"
```

## Production Considerations

- Prioritize network condition testing for flows involving user-entered data (forms, reviews, checkout) — silent data loss on a dropped connection is a significantly worse user experience than a slow load, and deserves proportionally more test coverage.
- Offline-first apps (or apps with local caching/sync) need dedicated testing around the sync/reconciliation logic when connectivity returns — this is often the most bug-prone part of network resilience, more so than the offline state itself.
- Network condition testing is a good candidate for a focused, separate test suite (rather than baking network simulation into every test) — most tests should run against stable, fast conditions for speed, with network resilience specifically tested in a dedicated, smaller set of scenarios.

## Common Pitfalls

- Only testing the fully-offline and fully-online states, skipping the transition scenarios (going offline mid-request, coming back online) — these transition points are where the most damaging bugs (silent data loss, duplicate submissions on retry) tend to live.
- Assuming a fast, stable CI network environment is representative of real mobile usage conditions — an app that "works fine" in CI can still fail badly for real users on unreliable networks if network resilience was never explicitly tested.
- Not verifying that user-entered data survives a failed submission — testing only that an error message appears, without checking whether the user's input was preserved for a retry.
- Treating "shows a loading spinner" as sufficient network handling — a spinner that never resolves (no timeout, no error state) on a genuinely failed request is itself a bug that needs a hard timeout/error path, not just a visual loading indicator.

## Interview Notes

- Be ready to describe how you'd test an app's behavior when connectivity drops mid-request — this specific scenario is a common, practical interview question that reveals whether a candidate thinks about transition states, not just static offline/online conditions.
- Understand why mobile apps need proportionally more network resilience testing than typical web apps, given real-world mobile connectivity patterns.
- Be able to describe a concrete example of graceful vs. ungraceful network failure handling (preserving user input on a failed submission, vs. silently losing it) — a strong, memorable example of what "good" network handling looks like in practice.

## References

- [Appium — Network Connection Control](https://appium.io/docs/en/latest/guides/network-connection/)
- [Android Developers — Testing on a Slow Connection](https://developer.android.com/develop/connectivity/testing-network-connections)