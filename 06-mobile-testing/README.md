# 06 — Mobile Testing

Native, hybrid, and mobile web testing — from Appium fundamentals to device farms, gestures, permissions, and mobile-specific performance concerns. Read [02-automation-python-playwright](../02-automation-python-playwright/) first — Appium's WebDriver-based API builds directly on those concepts.

## Notes

1. [Mobile Testing Fundamentals](./mobile-testing-fundamentals.md) — Emulators vs. real devices, device fragmentation, support matrices
2. [Mobile App Types & Testing Approach](./mobile-app-types-and-testing-approach.md) — Native vs. hybrid vs. mobile web, and how tooling differs
3. [Appium Fundamentals](./appium-fundamentals.md) — Architecture, desired capabilities, and the WebDriver-based API
4. [Mobile Locator Strategies](./mobile-locator-strategies.md) — Accessibility ID, platform-specific selectors, and locator priority
5. [Gestures & Touch Interactions](./gestures-and-touch-interactions.md) — Swipe, pinch, long-press via the W3C Actions API
6. [Mobile Device Farms & Cloud Testing](./mobile-device-farms-and-cloud-testing.md) — Scaling real-device coverage with BrowserStack/Sauce Labs-style services
7. [Mobile Permissions & Notifications Testing](./mobile-permissions-and-notifications-testing.md) — OS-level permission prompts and push notification flows
8. [Mobile Network Condition Testing](./mobile-network-condition-testing.md) — Offline, slow, and intermittent connectivity scenarios
9. [Mobile App Performance Testing](./mobile-app-performance-testing.md) — Launch time, memory, battery, and frame rate

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — the WebDriver/fixture concepts Appium testing builds on
- [01 — Manual Testing](../01-manual-testing/) — cross-platform compatibility testing principles this folder extends to mobile
- [14 — Performance Testing](../14-performance-testing/) — broader performance testing beyond mobile-specific client metrics