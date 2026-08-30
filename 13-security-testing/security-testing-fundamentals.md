# Security Testing Fundamentals

## What It Is

Security testing verifies that an application protects data and resources from unauthorized access, tampering, and disclosure. This note establishes the scope for this folder: the OWASP Top 10 as the standard reference for the most critical web application security risks, and — just as importantly — a clear line between what a QA/SDET can reasonably own as part of everyday functional testing versus what genuinely requires a dedicated security specialist or penetration tester.

## Why It Matters

- Security bugs are frequently more severe than functional bugs — a broken button inconveniences a user; a broken authorization check can expose every user's data (see [API Security Testing Basics](../04-api-testing/api-security-testing-basics.md) for this same point applied specifically to APIs).
- Most organizations don't have a security tester embedded in every team — a meaningful amount of security risk reduction comes from functional testers/SDETs catching foundational issues early, as a baseline layer beneath dedicated security testing, not a replacement for it.
- Interviewers ask about security testing awareness specifically to see whether a candidate's definition of "quality" extends beyond functional correctness — this is a common differentiator between QA Engineer and more senior SDET-level thinking (see [QA Engineer vs. SDET](../00-start-here/qa-engineer-vs-sdet.md)).

## How It Works

**The OWASP Top 10** is the industry-standard, periodically updated reference list of the most critical web application security risks, based on real-world data. The categories this folder covers in depth (injection, XSS, broken authentication/session management, CSRF, security misconfiguration, vulnerable dependencies) map directly to entries on this list — it's the organizing reference for the rest of this folder, not a topic covered once and set aside.

**What a functional SDET can reasonably own:**
- Basic negative/boundary testing that happens to catch injection-adjacent issues (see [Negative Testing for APIs](../04-api-testing/negative-testing-for-apis.md)).
- Verifying authentication/authorization boundaries are enforced correctly (see [API Security Testing Basics](../04-api-testing/api-security-testing-basics.md)'s BOLA testing).
- Checking that security headers are present and correctly configured (see [Security Headers & Configuration Testing](./security-headers-and-configuration-testing.md)).
- Running automated dependency vulnerability scans as part of CI (see [Dependency & Supply Chain Security Testing](./dependency-and-supply-chain-security-testing.md)).
- Basic, tool-assisted vulnerability scanning (see [Security Testing Tools Overview](./security-testing-tools-overview.md)).

**What genuinely requires a dedicated specialist:**
- Formal penetration testing and threat modeling.
- Deep exploitation research (finding genuinely novel vulnerability classes, not applying known checks).
- Compliance-driven security audits (PCI-DSS, SOC 2, etc.) requiring certified assessment.

## Example

A simple triage framework distinguishing what an SDET should test directly versus escalate to a specialist, applied to a hypothetical finding:

```text
Finding during routine functional testing: entering a single quote (')
in a search field causes a 500 error with a visible database stack
trace in the response.

SDET-level action (within reasonable scope):
  - This is EXACTLY the kind of finding a functional tester should
    catch and report — see Injection Attacks & Testing
  - File as a defect: unhandled input causing a server error AND
    leaking internal implementation details (see Security Headers &
    Configuration Testing on information leakage)
  - Flag as security-relevant, not just a generic bug, given the
    stack trace exposure

Escalation to a specialist (beyond reasonable SDET scope):
  - Determining whether this is actually EXPLOITABLE as a full SQL
    injection (constructing a working exploit chain) is a job for a
    dedicated security tester/pentester, not a routine part of
    functional QA
  - The SDET's job is to report the finding clearly and escalate,
    not to prove full exploitability themselves
```

## Production Considerations

- Establish a clear team norm for what gets escalated to a security specialist versus handled as a standard defect — without this, foundational security findings can get stuck in ambiguous ownership, with functional QA assuming security owns it and security assuming QA already handled it.
- Security testing awareness should be built into everyday functional testing (negative testing, boundary testing) rather than treated as a separate, occasional activity — many of the checks in this folder are extensions of testing discipline already covered elsewhere in this repo, not entirely new skills.
- Basic security testing (dependency scanning, header checks) is cheap to automate into CI (see [Dependency & Supply Chain Security Testing](./dependency-and-supply-chain-security-testing.md)) and should run continuously, not just during periodic manual review.

## Common Pitfalls

- Assuming security testing is entirely "someone else's job," missing foundational issues a functional tester is well-positioned to catch during normal testing work.
- The opposite mistake — attempting to perform deep exploitation/penetration testing without proper training or authorization, which can itself be legally and ethically fraught, not just technically risky.
- Treating the OWASP Top 10 as a one-time checklist to review once, rather than an ongoing reference informing everyday test design.
- Not having a clear escalation path for security-relevant findings, causing them to be under-prioritized as "just another bug" when they may need more urgent, specialized handling.

## Interview Notes

- Be ready to name several OWASP Top 10 categories and give a testable example of each — a common baseline security-awareness question.
- Understand and be able to articulate the boundary between what a functional SDET should own versus what needs specialist involvement — this shows mature judgment about scope, not just security knowledge in isolation.
- Be able to describe how you'd triage and escalate a security-relevant finding discovered during routine functional testing.

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)