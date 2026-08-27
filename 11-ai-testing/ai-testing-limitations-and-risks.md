# AI Testing Limitations & Risks

## What It Is

This note closes out the folder by consolidating the specific risk categories that apply across every AI-assisted testing technique covered so far: hallucination, over-trust, non-determinism, and privacy/security considerations of sending code or data to external AI tools. Where earlier notes covered technique-specific risks, this is the general risk framework worth applying to *any* new AI-assisted testing capability, including ones not yet covered elsewhere in this repo.

## Why It Matters

- AI capabilities in testing tools are evolving quickly, and new specific techniques will keep emerging beyond what any single repo can enumerate — having a general risk framework (rather than only technique-specific knowledge) is what lets an SDET evaluate a *new* AI testing capability critically, not just the ones already documented.
- Privacy/security risk specifically is often under-considered in testing contexts — sending application code, test data, or API responses to an external AI tool has real data-handling implications that many teams don't think through deliberately.
- Interviewers increasingly probe this specifically because uncritical AI adoption is a genuine, observed failure mode in the industry right now — being able to name specific risk categories (not just "be careful with AI") signals real, considered judgment.

## How It Works

**The four risk categories, applicable across any AI-assisted testing technique:**

1. **Hallucination** — an AI confidently generating plausible-but-false output (a nonexistent API method, a fabricated "expected result," an incorrect root-cause diagnosis) — covered technique-specifically in [AI-Assisted Test Automation Code](./ai-assisted-test-automation-code.md), but a general risk applicable anywhere AI generates specific factual claims.

2. **Over-trust** — treating AI output as more reliable than it actually is, skipping the human validation step that every note in this folder has emphasized — the single most common, avoidable risk across every technique covered.

3. **Non-determinism** — the same prompt can produce different output on different runs, meaning AI-generated test cases/code/analysis aren't fully reproducible the way a deterministic process would be — worth accounting for if relying on AI output as part of a repeatable process.

4. **Privacy & security** — sending proprietary application code, real (even if anonymized) data patterns, or internal API responses to an external AI tool has real data-handling implications: what does the tool's provider do with submitted data, is it used for further model training, does this violate any data-handling policy the organization has.

## Example

**A privacy/security consideration walked through concretely — evaluating whether to use an external AI tool on a specific piece of code:**

```text
Scenario: Considering pasting a full page object file (including
real API endpoint URLs and internal field names) into a public AI
chat tool to get help improving its locators.

Risk evaluation:
  - Does the organization have a policy on sending proprietary code
    to external AI tools? (Many do — check before assuming it's fine.)
  - Does the AI tool's terms of service specify whether submitted
    content is used for model training? (This varies significantly
    by tool and matters for anything containing real internal details.)
  - Is there a safer alternative — an enterprise/internal AI tool
    with different data-handling guarantees, or manually redacting
    internal URLs/identifiers before submitting to a public tool?

A reasonable resolution: redact real endpoint URLs and any internal-
only identifiers before submitting the code, or use an approved
internal tool if one exists — getting the same locator-improvement
help without the unconsidered data exposure.
```

**A non-determinism consideration, relevant to relying on AI output as part of a repeatable process:**
```text
Scenario: A team wants to use an AI tool to auto-generate a
"test coverage summary" comment on every PR.

Consideration: since the same underlying code diff could produce
SLIGHTLY different AI-generated summaries on different runs (non-
determinism), this is fine for a human-readable, advisory comment,
but would be a poor foundation for anything requiring exact
reproducibility (e.g., using AI output directly as a pass/fail
CI gate) — the non-deterministic nature needs to match the actual
tolerance of how the output will be used.
```

## Production Considerations

- Establish (or find out if the organization already has) a clear policy on what can and can't be shared with external AI tools — proprietary code, real data patterns, internal architecture details — before individual engineers make ad-hoc decisions about this on their own.
- For any AI-assisted process feeding into something requiring reproducibility or auditability (a documented test result, a compliance-relevant record), account for non-determinism explicitly — either by having a human confirm/finalize the output, or by not relying on AI output as the sole source of truth for anything requiring exact repeatability.
- Build "verify before trusting" into team workflow/culture as a default habit for AI-assisted testing work generally, rather than relying on each individual to remember it independently for each new tool/technique.

## Common Pitfalls

- Sending proprietary code, real data patterns, or internal architecture details to external AI tools without considering the organization's data-handling policies or the tool's own data-usage terms.
- Treating AI output as fully reproducible/deterministic when relying on it for something that actually requires that property (a documented, auditable process) — silently introducing inconsistency into a process that assumes consistency.
- Developing "AI fatigue" that swings to blanket distrust after being burned by hallucination once, missing the genuine, specific value AI assistance provides in well-suited contexts (covered throughout this folder).
- Not having any organizational policy or team norm around AI tool usage at all, leaving each individual to make ad-hoc, inconsistent privacy/security judgment calls.

## Interview Notes

- Be ready to name the general risk categories (hallucination, over-trust, non-determinism, privacy/security) that apply across any AI-assisted testing technique — a strong, organized answer versus a vague "AI can be wrong sometimes."
- Understand privacy/security risk specifically as a genuine, often-underconsidered concern in testing contexts, and be able to describe a concrete scenario (like the one above) showing how you'd evaluate it.
- Be able to describe how you'd approach an entirely new AI testing capability not covered by existing documentation, using this general risk framework — this demonstrates the framework's real, transferable value beyond memorizing specific technique risks.

## References

- [OWASP — LLM and Generative AI Security Solutions Landscape](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [ISTQB — AI Testing (Foundation Extension)](https://www.istqb.org/certifications/artificial-intelligence-tester)