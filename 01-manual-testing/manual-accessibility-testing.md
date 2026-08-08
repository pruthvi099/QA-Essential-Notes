# Manual Accessibility Testing

## What It Is

Manual accessibility testing verifies that an application can be used by people with disabilities — including those using screen readers, keyboard-only navigation, or who have visual, motor, or cognitive impairments. It complements automated accessibility scanning tools (like axe-core), which catch a meaningful subset of issues (missing alt text, contrast ratios, ARIA misuse) but structurally cannot judge whether a screen reader experience actually makes sense to a real user.

Accessibility testing is grounded in the **Web Content Accessibility Guidelines (WCAG)**, the standard most legal/compliance accessibility requirements reference.

## Why It Matters

- Automated accessibility tools typically catch only 30–50% of real accessibility issues (industry-cited estimates vary, but the gap is well documented) — things like "does this screen reader announcement make logical sense" or "is the tab order actually usable" require a human check.
- Accessibility is frequently a legal/compliance requirement (e.g., ADA in the US, EN 301 549 in the EU), not just a UX nicety — production accessibility gaps can carry real legal and financial risk, not only reputational.
- As an SDET, you're well positioned to add basic manual accessibility checks into exploratory sessions and to build automated a11y scanning into CI — understanding both halves (what automation catches vs. what needs a human) is what makes that coverage meaningful instead of a checkbox.

## How It Works

**WCAG is organized around four principles (POUR):**
- **Perceivable** — information must be presentable in ways users can perceive (text alternatives for images, sufficient color contrast, captions for video)
- **Operable** — interface components must be operable (keyboard-accessible, enough time to read/interact, no content that triggers seizures)
- **Understandable** — information and UI operation must be understandable (readable text, predictable navigation, input error identification)
- **Robust** — content must work with a wide range of assistive technologies (valid, semantic HTML/ARIA that screen readers can correctly interpret)

**Core manual checks an SDET can run without specialized accessibility training:**
1. **Keyboard-only navigation** — unplug the mouse; can you reach and operate every interactive element using Tab, Shift+Tab, Enter, and Space? Is focus visibly indicated at every step?
2. **Screen reader spot-check** — using a free screen reader (VoiceOver on macOS, NVDA on Windows), navigate key flows and check whether announcements make sense (e.g., a button announced as "button" with no label is a failure).
3. **Color contrast** — check text/background contrast against WCAG minimums (4.5:1 for normal text, 3:1 for large text) using browser dev tools or a contrast checker.
4. **Zoom/text resize** — zoom the browser to 200%; does content remain usable without horizontal scrolling or overlapping elements?

## Example

A manual accessibility spot-check log for a login form — the kind of lightweight session an SDET can run without deep specialist training:

```text
Manual Accessibility Check: Login Form

Keyboard Navigation:
  ✅ Tab reaches Email field, then Password field, then Login button,
     in logical order
  🐞 No visible focus indicator on the Password field (default browser
     outline appears to be overridden by CSS with no replacement) —
     keyboard users can't tell it's focused

Screen Reader (VoiceOver):
  ✅ Email field announced as "Email, edit text"
  🐞 Password field announced only as "edit text, secure" — no label
     read aloud; likely missing an associated <label> or aria-label
  🐞 "Show password" icon button announced as "button" with no
     accessible name — screen reader users can't tell what it does

Color Contrast:
  ✅ Body text: 4.8:1 (passes 4.5:1 minimum)
  🐞 Placeholder text in input fields: 2.1:1 (fails — placeholder text
     is often relied on as a label substitute, which is itself a
     separate issue to flag)

Zoom to 200%:
  ✅ Form remains usable, no overlapping elements
```

Findings translate directly into both bug reports and, where automatable, checks that belong in CI:

```python
# axe-core integration (via axe-playwright-python) catches SOME of
# the above automatically — e.g., contrast and missing labels —
# but NOT things like "does this announcement make logical sense"
from playwright.sync_api import Page
from axe_playwright_python.sync_playwright import Axe

def test_login_form_accessibility_scan(page: Page):
    page.goto("https://example.com/login")
    axe = Axe()
    results = axe.run(page)
    assert results.violations_count == 0, results.generate_report()
```

## Production Considerations

- Automated a11y scanning (axe-core or similar) should run in CI on every change as a baseline gate — it's fast and catches a real, meaningful subset of issues cheaply. Manual checks are reserved for new/changed flows and periodic deeper audits, not every commit.
- Accessibility should be considered during design/development, not only caught in testing — retrofitting keyboard navigation or proper ARIA semantics onto an already-built component is far more expensive than building it in from the start.
- For legal/compliance-driven accessibility requirements, a full audit by a specialist (or accessibility-focused QA) is typically still needed — basic SDET-level manual checks reduce risk but aren't usually a substitute for a formal WCAG conformance audit.

## Common Pitfalls

- Relying solely on automated scanning and assuming "0 violations" means the app is accessible — automated tools can't judge logical screen reader flow, meaningful alt text, or genuinely sensible tab order.
- Testing keyboard navigation only for "can I technically Tab to it" without checking focus visibility — an element that's technically reachable but has no visible focus indicator is still a real accessibility failure.
- Treating accessibility as a final pre-release check instead of an ongoing concern — this tends to produce a backlog of accessibility debt that's expensive to fix all at once.
- Using placeholder text as a substitute for actual form labels — placeholders disappear on input and often fail contrast requirements, a very common real-world finding.

## Interview Notes

- Be ready to describe a basic manual accessibility check you can run without specialized tools (keyboard-only navigation is the most commonly expected answer).
- Understand the POUR principles at a high level and be able to give one example finding per principle.
- Be able to explain clearly what automated a11y scanning catches vs. what still requires a human — this is a frequent, specific interview question that separates genuine accessibility awareness from a surface-level "we run axe-core" answer.

## References

- [W3C — Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/)
- [W3C — Understanding WCAG](https://www.w3.org/WAI/WCAG22/Understanding/)
- [Deque — axe-core](https://github.com/dequelabs/axe-core)