# Mobile App Types & Testing Approach

## What It Is

Mobile apps fall into three architectural categories — **native**, **hybrid**, and **mobile web** — and each demands a meaningfully different testing approach, tooling, and risk profile. Understanding which type an app is shapes nearly every downstream testing decision: which automation framework applies, what performance characteristics to expect, and where platform-specific bugs are likely to hide.

## Why It Matters

- Using the wrong tooling assumption for an app's type wastes real effort — trying to automate a hybrid app's WebView content with pure native locators (or vice versa) simply doesn't work, and misdiagnosing the app type early costs significant setup time.
- Each type has a different, predictable risk profile — native apps risk platform-specific bugs (iOS vs. Android divergence), hybrid apps risk WebView-native bridge issues, mobile web risks the same cross-browser fragmentation as desktop web (see [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md)) plus mobile-specific viewport/touch concerns.
- Interviewers ask "how would you test a hybrid app differently from a native one" specifically to check whether a candidate understands testing strategy is architecture-dependent, not one-size-fits-all.

## How It Works

| Type | What It Is | Built With | Testing Tooling |
|---|---|---|---|
| **Native** | Built specifically for one OS, using platform SDKs | Swift/Objective-C (iOS), Kotlin/Java (Android) | Appium, XCUITest (iOS), Espresso (Android) |
| **Hybrid** | A native shell wrapping web content (WebView) | React Native, Flutter, Ionic/Cordova + web tech | Appium (handles both native shell and WebView context switching) |
| **Mobile Web** | A responsive website accessed via a mobile browser, no app install | HTML/CSS/JS, same as desktop web | Playwright/Selenium with mobile viewport emulation, or real mobile browser testing |

**Key testing distinctions:**

- **Native** — fully platform-specific; iOS and Android versions of the "same" app are actually two separate codebases, meaning bugs can exist in one platform and not the other. Requires platform-specific automation (or a cross-platform tool like Appium abstracting both).
- **Hybrid** — the trickiest to automate correctly, since it involves *context switching* between the native app shell and an embedded WebView rendering web content — a locator strategy that works in the native context won't find WebView elements, and vice versa.
- **Mobile web** — closest to familiar web testing (see [02-automation-python-playwright](../02-automation-python-playwright/)), but still needs real mobile browser/device validation beyond just resizing a desktop browser viewport, since real mobile browsers have their own rendering quirks and touch-specific behavior.

## Example

Identifying app type from behavior, and the resulting testing implication — a practical diagnostic an SDET applies early on a new mobile project:

```text
Observation: The app has native-feeling navigation and animations, but
the "Help Center" section clearly renders as a web page (different
font rendering, a visible loading flash, scroll behavior that feels
like a browser).

Diagnosis: Likely a HYBRID app — native shell with an embedded WebView
for the Help Center specifically.

Testing implication: Automation needs to switch Appium's context
between NATIVE_APP and WEBVIEW when navigating in/out of the Help
Center — a single locator strategy won't work across both.
```

A context-switching example in Appium (Python), showing the hybrid-specific concern in code:

```python
from appium import webdriver

def test_help_center_hybrid_context_switch(driver):
    # Interacting with native app elements first
    driver.find_element("accessibility id", "help_center_button").click()

    # Switch context to interact with the WebView content
    contexts = driver.contexts
    webview_context = next(c for c in contexts if "WEBVIEW" in c)
    driver.switch_to.context(webview_context)

    # Now standard web-style locators work, since we're in the WebView
    driver.find_element("css selector", "h1.help-title").is_displayed()

    # Switch back to native context to continue interacting with the app shell
    driver.switch_to.context("NATIVE_APP")
    driver.find_element("accessibility id", "back_button").click()
```

## Production Considerations

- Identify the app's actual architecture (native, hybrid, or which hybrid framework — React Native vs. Flutter vs. Cordova) before choosing automation tooling — this is a foundational setup decision, and getting it wrong means reworking the framework later once the mismatch becomes apparent.
- For hybrid apps, coordinate with developers on which specific sections are WebView-based versus native — this isn't always obvious from using the app alone, and knowing it upfront avoids wasted debugging time on "locator not found" errors that are actually context-switching issues.
- Mobile web apps still need genuine mobile device/browser testing, not just a resized desktop browser — real mobile browsers (Safari on iOS, Chrome on Android) have their own rendering and touch-interaction behavior that viewport emulation alone doesn't fully replicate (see [Cross-Browser & Compatibility Testing](../01-manual-testing/cross-browser-compatibility-testing.md)).

## Common Pitfalls

- Assuming a single automation approach works for every screen in a hybrid app, without recognizing some screens are WebView-rendered and require context switching.
- Treating "iOS and Android versions of the app" as the same codebase during test planning — for native apps, they're genuinely separate implementations, and a passing test on one platform says nothing about the other.
- Testing a mobile web app only via desktop browser dev tools' responsive mode, skipping real mobile device/browser validation — this misses real touch-interaction and rendering differences.
- Not identifying which cross-platform framework (React Native, Flutter, etc.) a hybrid/cross-platform app uses — different frameworks have different Appium driver requirements and quirks worth knowing upfront.

## Interview Notes

- Be ready to explain the native/hybrid/mobile-web distinction clearly, with the testing tooling implication for each — a common, foundational mobile testing interview question.
- Understand Appium's context-switching concept for hybrid apps specifically — this is a frequently asked, practical detail that distinguishes real hybrid-app testing experience from surface-level knowledge.
- Be able to describe how you'd diagnose which type of app you're testing when it's not stated upfront — a practical, real-world scenario question.

## References

- [Appium Documentation — Automating Hybrid Apps](https://appium.io/docs/en/latest/guides/hybrid/)
- [Android Developers — App Architecture](https://developer.android.com/topic/architecture)