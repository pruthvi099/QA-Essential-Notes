# Cross-Browser & Compatibility Testing

## What It Is

Cross-browser and compatibility testing verifies that an application behaves and renders correctly across different browsers, browser versions, operating systems, screen sizes, and devices. It exists because browsers implement web standards (CSS, JavaScript APIs, rendering engines) with subtle differences, and users access the same application from a wide, uncontrolled variety of environments.

This note covers the manual approach; automated cross-browser execution (running the same Playwright/Selenium suite across multiple browsers) is covered separately once automation topics begin.

## Why It Matters

- A feature that works perfectly in the developer's browser (usually Chrome) can silently break in Safari, older browser versions, or specific mobile devices — these gaps often go unnoticed until real users report them, since the dev/QA team's own testing tends to cluster around whatever browser they personally use.
- Compatibility bugs are disproportionately costly in production: they're hard to reproduce ("works for me"), often invisible in server-side logs, and disproportionately affect specific user segments (e.g., older devices, specific regions with different dominant browsers).
- As an SDET, understanding *what actually varies* across browsers (not just "test in multiple browsers" as a vague instruction) determines whether your compatibility strategy — manual or automated — is actually targeted at real risk instead of arbitrary browser/device combinations.

## How It Works

**What typically varies across browsers/environments:**
- CSS rendering (flexbox/grid edge cases, font rendering, scrollbar behavior)
- JavaScript API support (newer APIs may be unsupported or behave differently in older browsers)
- Form control rendering (date pickers, dropdowns, file inputs look and behave differently per browser)
- Touch vs. mouse interaction handling (mobile Safari/Chrome vs. desktop)
- Viewport/responsive breakpoints on real devices vs. simulated ones

**Choosing what to cover** — testing every browser × version × OS × device combination is infeasible. Instead:
1. **Check real usage data** (analytics) to prioritize the browsers/devices your actual users use — don't guess.
2. **Define a support matrix** — explicitly state which browsers/versions/devices are officially supported, and which are "best effort" or unsupported.
3. **Prioritize by risk** — pages/flows with complex CSS layouts, custom form controls, or heavy JS interaction are higher risk than static content pages.

**Manual compatibility testing techniques:**
- Testing on real devices for at least the top 1–2 mobile browsers/OS combinations (emulators miss real touch/rendering quirks).
- Using browser dev tools' device emulation for a fast first pass across many viewport sizes, before deeper real-device testing on the highest-priority ones.
- Deliberately testing on the *oldest* browser version still in the support matrix, since that's where the most compatibility gaps surface.

## Example

A support matrix and a targeted manual compatibility check, structured for repeatability:

```text
Support Matrix — Product App

Tier 1 (fully supported, tested every release):
  - Chrome (latest 2 versions) — Desktop & Android
  - Safari (latest 2 versions) — Desktop & iOS

Tier 2 (best effort, tested pre-major-release):
  - Firefox (latest version) — Desktop
  - Edge (latest version) — Desktop
  - Samsung Internet — Android

Unsupported: Internet Explorer (any version)
```

```text
Compatibility Check: Checkout page, custom date picker (delivery date selector)

Chrome Desktop:     ✅ Renders correctly, keyboard nav works
Safari Desktop:     🐞 Date picker overflows below viewport on smaller
                        window widths — not visible without scrolling
Safari iOS:         ✅ Renders correctly, native date picker used
Firefox Desktop:    ✅ Renders correctly
Samsung Internet:   🐞 Touch targets for date cells are too small
                        (~28px, below the 44px recommended minimum),
                        hard to tap accurately
```

Findings like these become either bug reports (functional-adjacent, since they affect real usability per-browser) or, once fixed, candidates for automated cross-browser regression:

```python
# Once fixed, this becomes part of an automated cross-browser suite,
# run via Playwright's built-in multi-browser support
import pytest

@pytest.mark.parametrize("browser_name", ["chromium", "webkit", "firefox"])
def test_date_picker_visible_in_viewport(browser_name, playwright):
    browser = getattr(playwright, browser_name).launch()
    page = browser.new_page(viewport={"width": 768, "height": 600})
    page.goto("https://shop.example.com/checkout")
    page.click("#delivery-date-picker")

    picker = page.get_by_test_id("date-picker-panel")
    box = picker.bounding_box()
    assert box["y"] + box["height"] <= 600, "Date picker overflows viewport"
    browser.close()
```

## Production Considerations

- A support matrix should be a visible, agreed-upon artifact (shared with product/design), not an assumption — "we support IE11" vs. "we don't" changes CSS/JS decisions throughout development, not just testing scope.
- Automated cross-browser testing (Playwright/Selenium Grid, or cloud device farms like BrowserStack/Sauce Labs) should cover the *repeatable, well-defined* checks; manual testing stays valuable for real-device touch/rendering nuances that emulators and even automated cross-browser runs can miss.
- Compatibility issues found late (post-launch) are expensive to fix if the underlying CSS/JS approach wasn't compatibility-aware from the start — flagging likely compatibility risks (custom form controls, complex layouts) during design/code review is cheaper than finding them in testing.

## Common Pitfalls

- Testing only in the browser the tester personally prefers, missing the majority of real compatibility gaps entirely.
- Defining "cross-browser tested" vaguely without an actual support matrix — this leads to disputes about whether a bug in an obscure browser is even in scope.
- Relying solely on device emulation (browser dev tools) without any real-device testing — emulators don't accurately reproduce touch behavior, real GPU rendering, or OS-level UI quirks.
- Not prioritizing by usage data — spending equal effort on a browser with 0.5% of users as one with 40% is a poor use of limited testing time.

## Interview Notes

- Be ready to explain how you'd decide which browsers/devices to prioritize for compatibility testing — expect "we can't test everything, how do you choose?"
- Know the difference between emulated (dev tools) and real-device testing, and when each is sufficient.
- Be able to describe how compatibility testing evolves from manual (broad discovery) to automated (repeatable regression) once specific risk areas are identified — this connects the topic to your automation strengths.

## References

- [MDN — Cross Browser Testing](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Cross_browser_testing)
- [Playwright — Browsers](https://playwright.dev/docs/browsers)