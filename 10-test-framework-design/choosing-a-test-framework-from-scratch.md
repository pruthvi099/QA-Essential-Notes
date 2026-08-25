# Choosing a Test Framework From Scratch

## What It Is

This note covers the evaluation criteria for deciding what to build (or choose) when starting a new automation project from zero — language choice, tool choice (Playwright vs. Selenium vs. Cypress, etc.), test runner choice, and how much upfront architecture to invest in versus deferring until real needs emerge. It's a synthesis note, pulling together decision criteria touched on individually throughout the repo into one deliberate starting-point framework.

## Why It Matters

- Early framework decisions are expensive to reverse — a language or tool choice made in week one can still be shaping (and constraining) a team years later, making this a genuinely high-stakes decision worth deliberate evaluation rather than defaulting to "whatever I'm most familiar with."
- "What framework/tool would you choose for a new project and why" is one of the most common SDET interview questions specifically because it reveals whether a candidate can reason about trade-offs for a *specific context*, versus having one fixed, context-independent answer.
- This note ties together nearly every architectural concept from this folder — it's the practical entry point where all those individual decisions (layering, DI, config, utilities) first get made, even if only provisionally.

## How It Works

**Key evaluation dimensions:**

1. **Team's existing language expertise** — Python vs. TypeScript vs. others; leveraging existing team skill reduces ramp-up time significantly, and is often a stronger factor than any tool's individual feature set.
2. **Application type** — web (Playwright/Selenium/Cypress), mobile (Appium), API-only (language-native HTTP client + a test runner) — the application under test constrains tool choice more than personal preference should.
3. **Tool maturity and ecosystem** — active maintenance, community size, documentation quality, and how well it fits the application's specific needs (e.g., Playwright's native multi-browser support vs. needing separate tooling per browser).
4. **CI/CD integration needs** — how well a tool integrates with the team's existing pipeline infrastructure (see [07-ci-cd](../07-ci-cd/)) — a technically excellent tool that's awkward to run in the team's actual CI platform adds ongoing friction.
5. **Team size and growth trajectory** — a solo project or small team may not need the full layered architecture from day one (see [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md)); a large, growing team benefits from investing in structure earlier.

**How much upfront architecture to build:** start with the minimum structure that solves *today's* actual complexity, but choose patterns (POM, fixtures, basic config layering) that scale naturally rather than requiring a rewrite — the difference between deliberate minimalism and accidental disorganization is whether the simple starting structure was a conscious choice with a clear growth path, not just "we didn't think about it yet."

## Example

A structured evaluation for a hypothetical new project, showing the reasoning process rather than just a conclusion:

```text
Context: New e-commerce web application, team of 3 (2 with Python
backend experience, 1 with some JS/TS frontend experience), needs
CI integration with existing GitHub Actions pipelines, expects to
grow to a team of 8+ over the next year.

Evaluation:

Language: Python vs. TypeScript
  - 2 of 3 team members have strong Python experience; TypeScript
    experience is limited to 1 person
  - DECISION: Python + Playwright — leverages existing team strength,
    reduces ramp-up time significantly for the majority of the team
  - Trade-off acknowledged: TypeScript's Playwright Component Testing
    and slightly tighter Playwright Test integration are foregone,
    but not currently a priority need for this project

Tool: Playwright vs. Selenium vs. Cypress
  - Playwright: native multi-browser support, built-in parallelism,
    strong auto-waiting — matches this repo's earlier evaluation
    (see 02-automation-python-playwright/) of why Playwright is
    generally preferred for new projects today
  - DECISION: Playwright — modern, actively maintained, avoids
    Selenium's more manual wait/setup patterns

CI Integration: confirmed GitHub Actions support is first-class
  (see 07-ci-cd/github-actions-for-test-automation.md)

Architecture: given team WILL grow to 8+, invest in the layered
structure (BasePage, component objects, config layering) from the
START, rather than deferring — the growth trajectory justifies the
upfront investment that a smaller, stable team might reasonably skip
```

## Production Considerations

- Document the reasoning behind framework/tool choices (as in the evaluation above), not just the final decision — this context is valuable when the decision is questioned later, or when growth/changed circumstances genuinely warrant revisiting it.
- Revisit foundational choices periodically as context changes — a decision that was correct for a 3-person team may genuinely warrant reconsideration once the team has grown to 15, though this should be a deliberate, evidence-based reevaluation, not a reflexive "the grass is greener" response to a difficult day.
- Weight team familiarity heavily in the evaluation — the "objectively best" tool according to some external ranking is often not the right choice if it requires the whole team to ramp up from zero, especially under real delivery pressure.

## Common Pitfalls

- Choosing a tool/language based purely on personal preference or what's currently trendy, without weighing it against the team's actual existing expertise and the application's actual needs.
- Over-architecting a small, stable-team project with the full layered structure from day one when the added complexity isn't yet justified by real, current needs — matching the earlier caution in [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md).
- Under-architecting a project that's clearly going to grow significantly, leaving a costly restructuring effort for later instead of investing modestly upfront when the growth trajectory is already known.
- Not documenting the reasoning behind foundational choices, leaving future team members unable to distinguish a deliberate, considered decision from an arbitrary one when they're evaluating whether to revisit it.

## Interview Notes

- Be ready to walk through a structured evaluation for choosing a test framework/tool for a described hypothetical project — a very common, open-ended SDET interview question that rewards showing reasoning, not just naming a preferred tool.
- Understand that "it depends on context" is a legitimate, expected answer when properly followed by the actual dimensions that context would need to specify (team skill, application type, growth trajectory) — a vague "it depends" without substance is a weak answer, but a concrete list of decision criteria is strong.
- Be able to explain how much upfront architecture investment is appropriate for a given team size/trajectory, connecting back to [Test Framework Architecture Fundamentals](./test-framework-architecture-fundamentals.md)'s point about not over- or under-architecting.

## References

- [Playwright — Why Playwright (Node.js)](https://playwright.dev/docs/why-playwright)
- [ThoughtWorks Technology Radar — Test Automation Tools](https://www.thoughtworks.com/radar)