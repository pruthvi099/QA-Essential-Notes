# 15 — Accessibility & Visual Testing

Automated a11y scanning and visual regression at the tooling/CI level — extending the manual approaches from earlier folders with axe-core depth, WCAG conformance precision, and visual testing tool selection. Read [01-manual-testing](../01-manual-testing/) and [03-typescript-playwright](../03-typescript-playwright/) first for the manual accessibility and visual regression fundamentals this folder builds on.

## Notes

1. [Automated Accessibility Scanning](./automated-accessibility-scanning.md) — axe-core's outcome categories, impact levels, and real limits
2. [WCAG Conformance Levels](./wcag-conformance-levels.md) — A, AA, AAA, and why AA is the practical target
3. [Accessibility Testing in CI](./accessibility-testing-in-ci.md) — Severity-based gating without training bypass behavior
4. [Visual Testing Tools Comparison](./visual-testing-tools-comparison.md) — Playwright native vs. Percy vs. Chromatic vs. Applitools
5. [Cross-Device Visual Testing](./cross-device-visual-testing.md) — Catching viewport-specific regressions across breakpoints
6. [Combining Visual and Functional Assertions](./combining-visual-and-functional-assertions.md) — Choosing the right check for the right scenario
7. [Color Contrast & Visual Accessibility](./color-contrast-and-visual-accessibility.md) — The most common real-world a11y violation, in depth

## Related

- [01 — Manual Testing](../01-manual-testing/) — manual accessibility testing fundamentals this folder automates
- [03 — TypeScript Playwright](../03-typescript-playwright/) — the base visual regression pattern this folder extends
- [07 — CI/CD](../07-ci-cd/) — the quality-gate principles applied to a11y and visual checks here