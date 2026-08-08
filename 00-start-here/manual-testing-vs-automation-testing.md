# Manual Testing vs. Automation Testing

## What It Is

Manual testing is a human executing test steps and comparing actual vs. expected results without tool-driven execution. Automation testing is using a script/tool to execute those steps and assertions programmatically, without a human repeating them each time.

They are not competing approaches — they're complementary, each suited to different situations. Neither replaces the other entirely.

## Why It Matters

- One of the most common misconceptions entering this field is that automation "replaces" manual testing, or that SDET work is only about writing scripts. In reality, deciding *what not to automate* is as important a skill as automating.
- Interviewers frequently ask candidates to justify automation decisions — showing you understand the trade-offs (not just "automation is faster") signals engineering maturity.
- Misapplying automation (e.g., automating a one-time exploratory check) wastes more time than it saves, and misapplying manual testing (e.g., manually re-running the same regression suite every release) wastes far more.

## How It Works

| Factor | Favors Manual | Favors Automation |
|---|---|---|
| Frequency | Run once or rarely | Run repeatedly (every build/PR) |
| Stability | UI/requirements still changing | Stable, unlikely to change often |
| Nature | Exploratory, usability, ad-hoc | Repetitive, well-defined steps |
| Speed needed | Not time-critical | Needs fast feedback (CI/CD) |
| Human judgment | Needed (look-and-feel, UX) | Not needed (deterministic pass/fail) |
| ROI | Low volume, low repeat cost | High volume, automation pays off over time |

A simple rule of thumb: **automate when the cost of writing + maintaining the script is lower than the cumulative cost of repeating the test manually.**

## Example

The same regression check, shown both ways, to illustrate what "automatable" looks like:

**Manual test case:**
```text
Test Case: Verify user can add item to cart
Steps:
  1. Go to product page
  2. Click "Add to Cart"
  3. Open cart
Expected: Item appears in cart with correct price and quantity = 1
```

**Automated equivalent (Playwright/Python) — worth automating because it will run on every PR:**
```python
def test_add_item_to_cart(page):
    page.goto("https://shop.example.com/products/101")
    page.get_by_role("button", name="Add to Cart").click()
    page.get_by_role("link", name="Cart").click()

    cart_item = page.get_by_test_id("cart-item-101")
    assert cart_item.is_visible()
    assert cart_item.get_by_test_id("qty").inner_text() == "1"
```

By contrast, a one-off check like "does the new checkout page *feel* intuitive on a tablet" is exploratory/usability-driven and stays manual — there's no stable, deterministic assertion to automate.

## Production Considerations

- Mature teams keep a small, deliberate set of manual testing activities alongside automation: exploratory testing, usability testing, and testing genuinely new features before their behavior stabilizes enough to automate reliably.
- Automating too early (before requirements/UI stabilize) leads to constant script maintenance — a common cause of automation projects being abandoned.
- Automation has ongoing costs (maintenance, flaky test triage, infrastructure) that don't disappear after the initial script is written — this must be budgeted for, not treated as a one-time investment.

## Common Pitfalls

- Assuming 100% automation is the goal — some things (exploratory testing, first-pass usability checks) are structurally not well suited to automation.
- Automating unstable features early, causing scripts to break as often as they run and eroding trust in the suite.
- Treating manual testing as "lesser" work — exploratory manual testing frequently finds the bugs automation never will, because it isn't constrained by a predefined script.
- Not tracking automation ROI (time saved vs. maintenance cost), leading to bloated suites nobody trusts or prunes.

## Interview Notes

- Be ready to explain *when you would not automate something* — this is often a stronger signal than listing automation tools.
- Know how to argue ROI: frequency × manual execution time vs. automation build + maintenance cost.
- Understand that "100% automation" is rarely the actual goal of a mature QA organization — risk-based coverage is.

## References

- [ISTQB Foundation Level Syllabus](https://www.istqb.org/certifications/certified-tester-foundation-level)
- [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)