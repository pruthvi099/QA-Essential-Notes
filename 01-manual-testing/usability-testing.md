# Usability Testing

## What It Is

Usability testing evaluates how easy, intuitive, and efficient a system is for real users to accomplish their goals. Unlike functional testing (does it work?), usability testing asks whether it works *well for a human* — whether users can complete tasks without confusion, excessive effort, or errors caused by unclear design.

It's typically manual and judgment-based, since "is this confusing?" isn't a deterministic pass/fail check the way "does login redirect correctly?" is.

## Why It Matters

- A feature can pass every functional test and still fail in production because real users can't figure out how to use it — usability defects don't show up as crashes or wrong outputs, they show up as support tickets, drop-off rates, and negative reviews.
- As an SDET, you won't own usability testing the way a UX researcher does, but you're expected to recognize usability issues during exploratory sessions and know when to flag them separately from functional defects — they're a different class of problem with a different owner (design/product, not always engineering).
- Interviewers use usability questions to check whether your definition of "quality" extends beyond "does the code work" — a common signal of testing maturity.

## How It Works

Usability is commonly evaluated using **Jakob Nielsen's 10 usability heuristics** — general principles rather than pass/fail rules:

1. Visibility of system status (does the user know what's happening — loading states, progress indicators)
2. Match between system and the real world (familiar language/concepts, not technical jargon)
3. User control and freedom (can users undo/cancel/go back easily)
4. Consistency and standards (does the same action behave the same way everywhere)
5. Error prevention (does the design prevent mistakes before they happen)
6. Recognition rather than recall (don't force users to remember information across screens)
7. Flexibility and efficiency of use (shortcuts for experienced users, without complicating it for beginners)
8. Aesthetic and minimalist design (no irrelevant or rarely-needed information cluttering the interface)
9. Help users recognize, diagnose, and recover from errors (clear, actionable error messages)
10. Help and documentation (available when needed, without being required for basic use)

**Common usability testing methods:**
- **Heuristic evaluation** — an evaluator reviews the interface against the 10 heuristics above, without needing real end users.
- **Think-aloud testing** — a real user attempts tasks while verbalizing their thought process, revealing confusion points in real time.
- **First-click testing** — measuring whether users click the correct element first when attempting a task, a strong predictor of overall task success.

## Example

A heuristic evaluation applied to a checkout flow, structured as findings (this is the practical output format, not a pass/fail test case):

```text
Feature: Checkout page

Heuristic: Visibility of System Status
  Finding: After clicking "Place Order," there is no loading indicator
  for ~3 seconds. Users may click the button multiple times, unsure
  if the click registered.
  Severity: Medium (usability) — recommend a loading spinner + disabled
  button state during submission.

Heuristic: Error Prevention
  Finding: The "Remove item" button has no confirmation step and no
  undo. A single misclick removes an item with no recovery.
  Severity: Medium — recommend an "Item removed [Undo]" toast, or a
  confirmation step.

Heuristic: Recognition Rather Than Recall
  Finding: The order summary (items, total) is only shown on step 1;
  by step 3 (payment), users can no longer see what they're paying for
  without navigating back.
  Severity: High — recommend a persistent order summary sidebar across
  all checkout steps.
```

Note this isn't a script with "expected result" — it's structured *findings*, since usability issues are evaluated against principles, not fixed assertions. That said, once a usability finding leads to a concrete fix (e.g., "show a loading spinner"), it becomes testable functionally:

```python
def test_place_order_shows_loading_state(page):
    page.goto("https://shop.example.com/checkout")
    page.click("#place-order-btn")

    # The usability fix (loading state) is now a functional, automatable check
    assert page.get_by_test_id("order-loading-spinner").is_visible()
    assert page.get_by_role("button", name="Place Order").is_disabled()
```

## Production Considerations

- Usability testing is most valuable early (on prototypes/mockups) and on new flows — fixing a usability issue after launch costs more (redesign, re-testing, user re-training) than catching it before development.
- Usability findings should be logged with severity like functional defects, but tracked separately or tagged distinctly — they usually don't block a release the way a functional Critical defect does, but they matter for product quality trends over time.
- Once a usability fix introduces a concrete, checkable behavior (a loading state, an undo option, a confirmation dialog), that specific behavior becomes automatable — automation should pick up from where usability testing identifies a fix, not attempt to automate "is this intuitive" itself.

## Common Pitfalls

- Treating usability issues as bugs with the same severity/priority scale as functional defects — a confusing button label is not equivalent to a checkout crash, even though both are "issues."
- Skipping usability testing because "the QA team isn't UX" — basic heuristic evaluation during exploratory