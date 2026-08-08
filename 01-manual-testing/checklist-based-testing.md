# Checklist-Based Testing

## What It Is

Checklist-based testing uses a list of items to verify — without the full detail of a structured test case (no fixed steps, no specific expected result per item, no test data field). Each item is typically a short reminder of *what* to check, and the tester decides *how* to check it in the moment, relying on their own knowledge of the system.

It sits between ad-hoc and structured testing: more disciplined than ad-hoc (there's a defined list, so nothing obvious gets forgotten), but far faster to write and execute than full structured test cases.

## Why It Matters

- Checklists are the practical tool for high-frequency, low-detail verification — pre-release sanity checks, code review standards, deployment verification — where writing full test cases for each item would be disproportionately slow.
- As an SDET, checklists show up constantly outside pure "testing" too: PR review checklists, release checklists, on-call runbook checks — knowing how to write an effective one is a broadly reusable skill.
- Interviewers sometimes ask you to build a quick checklist on the spot (e.g., "what would you check before signing off on this release?") — it's a fast way to assess whether you think in terms of risk coverage, not just individual features.

## How It Works

An effective checklist:
- Lists items as specific, verifiable statements — not vague topics.
- Is ordered by risk/priority when execution time is limited (most important items first).
- Is scoped to a clear purpose (e.g., "pre-release sanity," not "test everything").
- Is short enough to actually get used consistently — a 100-item checklist gets skipped under time pressure; a focused 10–15 item one gets run every time.

Checklists work well for:
- **Pre-release sanity checks** — is the build minimally viable to test/release further?
- **Cross-cutting concerns** — things that apply to every feature (does it work on mobile, does it log errors correctly, are there console errors).
- **Code/PR review** — a consistent bar every change is checked against.
-