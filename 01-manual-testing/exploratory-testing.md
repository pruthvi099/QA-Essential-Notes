# Exploratory Testing

## What It Is

Exploratory testing is a manual testing approach where test design and test execution happen simultaneously — the tester explores the application, learns its behavior in real time, and uses that learning to guide the next steps, rather than following a pre-written, fixed script. It's structured around a mission or charter, not scripted step-by-step cases.

This is distinct from ad-hoc testing (random, undirected poking around) — exploratory testing is disciplined and goal-directed, even though it's not scripted.

## Why It Matters

- Scripted tests (manual or automated) only find what they were explicitly written to check. Exploratory testing finds the unknown unknowns — bugs nobody thought to write a test case for.
- It's one of the testing activities that automation genuinely cannot replace, because it relies on human judgment, curiosity, and adaptive reasoning about what "looks wrong."
- Even in highly automated teams, exploratory testing remains a core skill for SDETs — it's often used to test brand-new features before their behavior is stable enough to automate, and to probe automation blind spots.

## How It Works

Exploratory testing is typically organized using **session-based test management (SBTM)**:

1. **Charter** — a short mission statement defining what to explore (e.g., "Explore the checkout flow's handling of invalid/expired discount codes").
2. **Time-boxed session** — usually 30–90 minutes of focused, uninterrupted testing.
3. **Note-taking in real time** — record what was tested, what was observed, questions raised, and bugs found (not a rigid script, but a running log).
4. **Debrief** — a short session summary shared with the team: coverage achieved, defects found, risk areas identified, and follow-up charters needed.

Common techniques used *during* an exploratory session:
- **Boundary probing** — trying edge values without a predefined test case.
- **Error guessing** — using experience/intuition to try inputs likely to break the system (special characters, extremely long strings, rapid double-clicks).
- **Persona-based exploration** — testing as if you were a specific type of user (a first-time user, a power user, someone on a slow connection).

## Example

A sample session charter and resulting note log:

```text
Charter: Explore discount code entry at checkout, focused on invalid/
         edge-case inputs. Time-box: 45 minutes.

Session Notes:
  10:02 - Entered expired code "SUMMER23" → correctly rejected with
          clear message. ✅
  10:07 - Entered code with trailing whitespace " SAVE10 " → accepted
          silently, discount NOT applied, no error shown. 🐞 BUG
  10:15 - Entered code in lowercase "save10" (valid code is uppercase)
          → rejected. Question: is this intended case-sensitivity?
          Raised with PM.
  10:22 - Rapidly clicked "Apply" 5x on a valid code → discount applied
          5 times, total went negative. 🐞 BUG (likely missing
          idempotency/debounce handling)
  10:30 - Tried code field with emoji input → input silently truncated,
          no crash, acceptable behavior.

Session Summary:
  Coverage: Discount code entry, focused on invalid/whitespace/rapid-click
            scenarios
  Bugs found: 2 (whitespace handling, double-apply race condition)
  Open question: intended case-sensitivity behavior — needs PM clarification
  Follow-up charter suggested: explore discount code interaction with
  cart item removal mid-checkout
```

Bugs like the rapid-click race condition are a strong example of what exploratory testing catches that a scripted test (manual or automated) likely wouldn't, unless someone had already anticipated that exact scenario.

## Production Considerations

- Exploratory testing is most valuable on new/changed features, complex interaction points, and after automated regression has already passed — use it to complement automation, not duplicate what's already scripted.
- Session notes should feed back into the test suite: real bugs found exploratorily (like the double-apply race condition above) often become new automated regression tests once fixed, so the same class of bug doesn't resurface silently.
- SBTM output (charters, session notes, debriefs) should be lightweight and fast to write — if the note-taking overhead becomes heavy, testers stop doing it properly and the technique loses its value.

## Common Pitfalls

- Confusing exploratory testing with unstructured "ad-hoc" testing — without a charter and time-box, sessions produce inconsistent coverage and no reusable record.
- Treating exploratory testing as something only done when there's no automation — it should be a deliberate, planned activity even in a mature automated suite.
- Not converting exploratory findings into either automated tests or bug reports — insights get lost if they live only in a tester's head or an informal chat message.
- Skipping exploratory testing on "already automated" features — automation only tests what it was written to test; it doesn't replace human judgment about new edge cases.

## Interview Notes

- Be ready to explain the difference between exploratory and ad-hoc testing precisely — interviewers often use this to check if you understand exploratory testing as a *disciplined* technique.
- Know what session-based test management (SBTM) is and its core artifacts (charter, session notes, debrief) — this shows exploratory testing experience beyond just "I click around a lot."
- Be ready to describe a real (or plausible) bug you'd expect exploratory testing to catch that scripted/automated testing likely wouldn't.

## References

- [ISTQB Foundation Level Syllabus — Exploratory Testing](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [James Bach — Session-Based Test Management](https://www.satisfice.com/session-based-test-management)