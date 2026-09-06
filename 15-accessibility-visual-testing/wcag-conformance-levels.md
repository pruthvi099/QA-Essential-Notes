# WCAG Conformance Levels

## What It Is

WCAG (Web Content Accessibility Guidelines) defines three conformance levels — **A**, **AA**, and **AAA** — each success criterion assigned to exactly one level, and conformance is cumulative: meeting AA means satisfying all A and AA criteria. This note covers what each level actually requires, which one teams realistically target, and why — extending the POUR-principle overview from [Manual Accessibility Testing](../01-manual-testing/manual-accessibility-testing.md) with the specific conformance-level structure that governs legal and organizational targets.

## Why It Matters

- "WCAG compliant" is a meaningless claim without specifying a level and version — nearly every accessibility law, regulation, and legal settlement that references WCAG specifically means **Level AA** (most commonly WCAG 2.1 or 2.2 AA), not just "some undefined amount of WCAG."
- Knowing precisely what separates A from AA (and why AAA is rarely targeted site-wide) is directly practical for scoping accessibility testing effort and setting a realistic, defensible target with stakeholders.
- This is a frequently and precisely tested interview topic — being able to state exactly why AA (not A, not AAA) is the standard target, with the reasoning, is a stronger answer than just naming the three levels.

## How It Works

**Level A (minimum)** — the most basic accessibility requirements; satisfying only Level A leaves significant, common barriers unaddressed and will not satisfy any major accessibility law on its own — it's the floor, not a realistic target.

**Level AA (the practical target for the vast majority of organizations)** — addresses the most common barriers and is the level referenced by the majority of accessibility laws, regulations, and legal settlements (Title II ADA state/local government requirements, the EU's European Accessibility Act, EN 301 549). When legislation or a contract requires "WCAG compliance" without further qualification, it almost always means AA.

**Level AAA (highest)** — includes all A and AA criteria plus additional, stricter criteria (enhanced contrast ratios, extended audio descriptions for video, cognitive-accessibility-focused criteria). The W3C itself states AAA conformance should not be required as general policy for entire sites, since it's not possible to satisfy every AAA criterion for all types of content — AAA is realistically pursued only for specific content or specific criteria, not site-wide.

**Version matters alongside level** — WCAG 2.0 (2008), 2.1 (2018), and 2.2 (October 2023) each added new success criteria (2.1 added mobile/touch and low-vision criteria; 2.2 added focus visibility, minimum target size of 24×24 CSS pixels, and accessible authentication, while removing one obsolete criterion) — "WCAG AA" without a version number is itself an incomplete claim.

## Example

A concrete before/after showing how conformance level determines whether a specific finding is even in scope:

```text
Finding: A "Skip to main content" link is missing from the page.
  → This is Success Criterion 2.4.1 (Bypass Blocks), Level A.
  → Missing this fails even the MINIMUM conformance level.

Finding: Color contrast for body text is 4.2:1 (below the 4.5:1
minimum for normal text).
  → This is Success Criterion 1.4.3 (Contrast Minimum), Level AA.
  → An application targeting only Level A would NOT be required to
    fix this; an application targeting AA (the realistic standard) MUST.

Finding: Color contrast for body text is 6.8:1 (meets AA's 4.5:1,
but not AAA's stricter 7:1 requirement for normal text).
  → This is Success Criterion 1.4.6 (Contrast Enhanced), Level AAA.
  → An AA-targeting application is NOT required to fix this — this
    is exactly the kind of finding correctly deprioritized when AA,
    not AAA, is the agreed target.
```

A documented, precise conformance target — the kind of statement that should exist for any real project, extending [Entry/Exit Criteria & Test Metrics](../00-start-here/entry-exit-criteria-and-test-metrics.md)'s principle of explicit, agreed criteria to accessibility specifically:
```text
Accessibility Conformance Target: WCAG 2.2, Level AA, site-wide.

Level AAA criteria are NOT required, except: 1.4.6 (Contrast
Enhanced) is additionally targeted for the checkout flow specifically,
per a deliberate product decision to exceed the baseline for the
highest-risk, revenue-critical flow.
```

## Production Considerations

- State the conformance target precisely — level AND version (e.g., "WCAG 2.2 AA") — as an explicit, documented, cross-team standard, the same way a support matrix or entry/exit criteria are documented elsewhere in this repo, rather than leaving "accessible enough" as a vague, undefined aspiration.
- When legal/regulatory requirements apply (public sector, EU consumer-facing services under the European Accessibility Act from June 2025), confirm the *specific* required version and level with legal/compliance stakeholders — requirements vary by jurisdiction and sector, and assuming a default without confirming is a real business risk.
- Selectively targeting specific AAA criteria for specific high-value flows (as in the example) is a reasonable, deliberate choice — but should be an explicit decision, not an accidental byproduct of over-applying automated tooling defaults.

## Common Pitfalls

- Claiming "WCAG compliant" without specifying level and version — this is an incomplete, effectively meaningless claim that invites disputes about what was actually promised.
- Assuming Level A is a sufficient target — it is the floor and will not satisfy any major accessibility law on its own; AA is where legal and practical expectations actually sit.
- Setting AAA as a blanket, site-wide target — the W3C itself advises against this, since it's genuinely not achievable for all content types, and pursuing it uniformly wastes effort relative to a properly scoped AA target with selective AAA enhancements where they matter most.
- Treating WCAG version as interchangeable — "WCAG AA" without specifying 2.0/2.1/2.2 obscures which specific success criteria are actually in scope, since each version added new ones.

## Interview Notes

- Be ready to state precisely why Level AA (not A, not AAA) is the practical, legally-referenced target for the vast majority of organizations — this specific reasoning is what interviewers are checking for, not just the ability to name three levels.
- Understand that conformance is cumulative (AA includes A; AAA includes A and AA) and be able to give a specific example of a criterion at each level.
- Be able to explain why AAA is deliberately not recommended as a blanket, site-wide target, citing the W3C's own guidance — this shows precise, sourced understanding rather than a general impression.

## References

- [W3C — WCAG 2 Level AA Conformance](https://www.w3.org/WAI/WCAG2AA-Conformance)
- [W3C — Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/)