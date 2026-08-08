# Ad-Hoc vs. Structured Testing

## What It Is

Both are manual testing approaches, but they sit at opposite ends of a discipline spectrum:

- **Structured testing** — execution follows pre-written, detailed test cases with defined steps, data, and expected results (see [Writing Effective Test Cases](./writing-effective-test-cases.md)).
- **Ad-hoc testing** — informal, unplanned testing with no predefined test cases, no charter, and no fixed structure — the tester tries things based on intuition in the moment.

This is easy to confuse with [Exploratory Testing](./exploratory-testing.md), which is *also* unscripted — the key difference is that exploratory testing is disciplined (charter, time-box, logged notes), while ad-hoc testing is not.

## Why It Matters

- Ad-hoc testing gets a bad reputation because it's often confused with exploratory testing, or dismissed as "not real testing" — understanding exactly where it fits (and its real limitations) prevents both overuse and unfair dismissal.
- Knowing when structured testing is necessary (compliance, complex regulated flows, onboarding new testers) vs. when it's overkill (quick sanity check, familiar simple feature) is a practical judgment call SDETs make constantly under time pressure.
- Interviewers use this comparison to check whether you understand the *trade-offs* of unscripted testing, not just that "scripted is professional, unscripted is unprofessional" — that's an oversimplification that misses exploratory testing's real value.

## How It Works

| Aspect | Structured Testing | Ad-Hoc Testing | Exploratory Testing |
|---|---|---|---|
| Pre-defined steps | Yes, detailed | No | No |
| Planning | Full test case design upfront | None | Charter/mission defined upfront |
| Documentation during execution | Following existing test case | Usually none | Real-time session notes |
| Repeatability | High — same case, same result | Low — no record of what was tried | Medium — session notes allow reconstruction |
| Best suited for | Regression, compliance-critical flows, onboarding new testers | Quick, low-stakes sanity checks | Finding unknown defects in complex/new features |
| Risk | Time-consuming to write/maintain | Coverage gaps, nothing traceable if a bug isn't found | Requires skill/experience to be effective |

The spectrum, roughly:

```text
Fully Scripted -------- Exploratory (charter + notes) -------- Ad-Hoc (no structure)
Structured Testing         Disciplined, goal-directed            Unplanned, informal
```

## Example

The same feature area, approached three ways, to make the distinction concrete:

**Structured:**
```text
Test Case ID: TC_SEARCH_014
Steps: 1. Enter "laptop" in search box  2. Press Enter
Expected: Results page shows items containing "laptop" in title/description,
          sorted by relevance by default
```

**Ad-hoc** (no plan, no record — just trying things):
```text
[Tester types random queries into the search box for a few minutes,
 including "laptop", "asdkjh", an emoji, and a very long string, without
 taking notes on what was tried or found. If nothing looks wrong,
 nothing is reported. If something is found, it's reported — but there's
 no record of everything else that WAS tried.]
```

**Exploratory** (charter + real-time notes — see [Exploratory Testing](./exploratory-testing.md) for the full technique):
```text
Charter: Explore search behavior with unusual/edge-case queries.
         Time-box: 30 minutes.

Notes:
  - "laptop" → correct results, relevance-sorted ✅
  - "" (empty query) → shows ALL products instead of an error or
    empty state. Question: intended behavior? 🐞 Flagged for review
  - Very long string (500 chars) → search hangs for ~8 seconds before
    returning empty results. 🐞 Possible performance issue, filed as bug
  - Emoji input → silently ignored, empty state shown correctly ✅
```

Only the exploratory version produces a reusable, traceable record of what was actually covered — the ad-hoc version found nothing here, but there's no way to know if that's because nothing was