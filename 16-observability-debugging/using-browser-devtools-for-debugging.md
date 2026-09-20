# Using Browser DevTools for Debugging

## What It Is

This note covers browser developer tools — the Network tab, Console, and Application/Storage panels — as a client-side complement to the backend observability pillars covered in [Observability Fundamentals](./observability-fundamentals.md). Where logs/metrics/traces reveal what's happening on the server, DevTools reveals what's happening in the browser itself: the actual requests being made, JavaScript errors, and client-side storage state.

## Why It Matters

- A meaningful class of bugs lives entirely on the client side (a JavaScript error preventing a button from working, a request never actually being sent, stale cached data in local storage) — these are invisible to backend logs/traces entirely, since the backend never even receives a request in some of these cases.
- DevTools fluency is foundational, practical, everyday debugging skill — distinct from Playwright's own debugging tools (see [Debugging & VS Code Integration](../03-typescript-playwright/debugging-and-vscode-integration.md)), which are for debugging *test code*; DevTools is for debugging the *application itself*, live, as a real user would experience it.
- This is often assumed baseline knowledge in interviews rather than explicitly asked about — but being able to describe a specific DevTools-based investigation clearly is a strong, concrete signal when the opportunity arises.

## How It Works

**Network tab** — shows every HTTP request the page makes: URL, method, status code, request/response headers, and body. This is the client-side equivalent of the request/response validation covered in [Request & Response Validation](../04-api-testing/request-response-validation.md), but observed live, exactly as the browser actually sent/received it.

**Console** — shows JavaScript errors, warnings, and any explicit `console.log()` output — often the very first place a client-side bug reveals itself, before any backend investigation is even relevant.

**Application/Storage panel** — inspects cookies, local storage, session storage, and IndexedDB — useful for diagnosing bugs related to client-side state (a stale cached value, a missing/expired auth token, a feature flag stuck in an old state).

**A key investigative principle:** before assuming a bug is a backend issue, check whether the request was even *sent correctly* (or sent at all) via the Network tab — this immediately distinguishes a client-side bug (request malformed or never sent) from a genuine backend issue (request was sent correctly, but the response was wrong).

## Example

**A realistic investigation distinguishing a client-side bug from a backend issue, using DevTools as the first diagnostic step:**

```text
Bug report: "Clicking 'Apply Discount' does nothing — no error shown,
no discount applied."

Step 1 — Console tab check:
  Uncaught TypeError: Cannot read properties of undefined (reading 'value')
    at applyDiscount (checkout.js:142)

  This IMMEDIATELY reveals the bug is CLIENT-SIDE — a JavaScript
  error is preventing the click handler from completing, meaning
  the API request was likely never even sent. No backend
  investigation needed at all; this is a frontend code bug.

Step 2 — Confirming via Network tab:
  Filtering for the expected /api/orders/discount request shows
  NOTHING was sent when the button was clicked — confirming the
  JavaScript error above prevented the request from firing entirely,
  consistent with the Console finding.
```

**A contrasting scenario where DevTools correctly points toward a backend issue instead:**
```text
Bug report: "Discount doesn't apply, but no error is shown."

Step 1 — Console tab: no JavaScript errors present.

Step 2 — Network tab: the request WAS sent correctly:
  POST /api/orders/discount
  Request payload: {"code": "SAVE10", "order_id": 501}
  Status: 200 OK
  Response: {"discount_applied": false, "reason": "order_total_below_minimum"}

  The request/response cycle worked correctly — the API responded
  as designed, just with a result the UI failed to communicate to
  the user. This is a DIFFERENT bug: either a UX gap (no error
  message shown for a legitimate rejection) or a backend LOGIC
  issue (why was the order below the minimum when it shouldn't be) —
  and NOW backend logs/traces (per Reading Application Logs) are
  the right next step, since DevTools has already confirmed the
  client-side request/response cycle itself worked correctly.
```

**Inspecting stale client-side state via the Application panel:**
```text
Bug: A user reports seeing an old, outdated feature flag state
after a known rollout.

Application panel → Local Storage → checking the stored
feature-flags value shows a STALE cached object from before the
rollout, never refreshed — explaining the bug without needing any
backend investigation at all, since the backend is correctly
returning the NEW flag state; the client just never re-fetched it.
```

## Production Considerations

- Checking the Console and Network tab first, before any backend investigation, is a fast, low-cost triage step that immediately narrows whether a bug is client-side or backend — worth making a standard first move for any reported UI bug, not an occasional afterthought.
- When filing a defect for a client-side bug found via DevTools, include the specific console error/stack trace and relevant network request details directly in the report (see [Defect Reporting Best Practices](../01-manual-testing/defect-reporting-best-practices.md)) — this is exactly the kind of concrete evidence that makes a report immediately actionable.
- DevTools' Network tab can also be used to manually replay/modify a request (via "Copy as cURL" or similar), a quick, ad-hoc way to isolate whether an issue is reproducible independent of the UI at all.

## Common Pitfalls

- Jumping straight to backend log investigation for a bug that's actually client-side (a JavaScript error preventing a request from ever being sent) — checking the Console first would have revealed this immediately and saved significant investigation time.
- Not checking the Network tab to confirm whether a request was actually sent (and sent correctly) before assuming a backend response is wrong — sometimes the "backend bug" is actually a malformed or missing client-side request.
- Overlooking client-side storage (local storage, session storage) as a source of a bug, when a stale cached value can produce symptoms that look identical to a real backend inconsistency.
- Filing a defect report based only on the visible symptom ("button doesn't work") without checking DevTools first, missing the specific, immediately actionable detail (a console error, a failed network request) that would make the report far more useful.

## Interview Notes

- Be ready to describe how you'd use DevTools to triage whether a reported bug is client-side or backend — the Console-then-Network-tab sequence is a strong, concrete, practical answer.
- Understand what each panel (Network, Console, Application/Storage) is specifically useful for, and be able to map a described symptom to the panel most likely to reveal the cause.
- Be able to give a concrete example (like the JavaScript error scenario above) of a bug that DevTools reveals instantly but backend log investigation would have missed or taken much longer to find.

## References

- [Chrome DevTools — Documentation](https://developer.chrome.com/docs/devtools/)
- [MDN — Firefox Developer Tools](https://developer.mozilla.org/en-US/docs/Tools)