# HTTP Methods & Status Codes

## What It Is

HTTP methods (verbs) define the type of action a request performs on a resource, and status codes communicate the result of that request. API testing fundamentally means verifying that a given method against a given endpoint produces the correct status code and response for both valid and invalid inputs — this note covers the semantics that make those expectations correct in the first place.

## Why It Matters

- Misunderstanding method semantics (e.g., treating PUT and PATCH as interchangeable) leads to writing tests that verify the wrong behavior, or missing bugs where an API violates its own method's contract (a GET that has side effects, a POST that isn't actually creating a new resource).
- **Idempotency** — whether repeating the same request produces the same result — is a property testers must specifically verify, not assume; it's one of the most common sources of subtle production bugs (e.g., a non-idempotent PUT causing duplicate side effects on retry).
- This is foundational, near-universal interview material for any API testing role — being precise about method semantics and status code categories is an easy, expected baseline that's still commonly gotten wrong.

## How It Works

**HTTP Methods:**

| Method | Purpose | Idempotent? | Safe (no side effects)? |
|---|---|---|---|
| GET | Retrieve a resource | Yes | Yes |
| POST | Create a resource / trigger an action | No | No |
| PUT | Replace a resource entirely | Yes | No |
| PATCH | Partially update a resource | No (typically) | No |
| DELETE | Remove a resource | Yes | No |

**Idempotent** means calling it multiple times has the same effect as calling it once (GET, PUT, DELETE). **Safe** means it doesn't change server state at all (GET only). Testing idempotency specifically means: call the same request twice, verify the resulting state is identical to calling it once — not just that both calls individually "succeeded."

**Status Code Categories:**

| Range | Category | Common codes |
|---|---|---|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

A frequently confused pair: **401 Unauthorized** means "you're not authenticated" (missing/invalid credentials); **403 Forbidden** means "you're authenticated, but not allowed to do this" (valid credentials, insufficient permissions). Testing both distinctly is a common, specific check interviewers ask about.

## Example

Testing method semantics and idempotency directly, using Playwright's `APIRequestContext` (see [API Testing with Playwright](../02-automation-python-playwright/api-testing-with-playwright.md)):

```python
def test_put_is_idempotent(api_context):
    payload = {"name": "Updated Product Name", "price": 999}

    # Call PUT twice with the identical payload
    first_response = api_context.put("/api/products/101", data=payload)
    second_response = api_context.put("/api/products/101", data=payload)

    assert first_response.status == 200
    assert second_response.status == 200

    # The resulting state should be IDENTICAL after either call —
    # this is what actually verifies idempotency, not just that
    # both calls returned 200
    assert first_response.json() == second_response.json()

def test_post_is_not_idempotent(api_context):
    payload = {"name": "New Product", "price": 500}

    first_response = api_context.post("/api/products", data=payload)
    second_response = api_context.post("/api/products", data=payload)

    # Each POST creates a NEW resource with a different ID —
    # calling it twice should NOT produce the same result
    assert first_response.json()["id"] != second_response.json()["id"]

def test_401_vs_403_distinction(api_context):
    # No auth token at all
    no_auth_response = api_context.get("/api/admin/users", headers={})
    assert no_auth_response.status == 401

    # Valid token, but insufficient permissions (non-admin user)
    insufficient_perms_response = api_context.get(
        "/api/admin/users",
        headers={"Authorization": "Bearer valid-standard-user-token"}
    )
    assert insufficient_perms_response.status == 403
```

## Production Considerations

- Test idempotency explicitly for any endpoint your application relies on being safely retryable (common in distributed systems with automatic retry logic) — an API that claims to be idempotent but isn't can cause real production incidents (duplicate charges, duplicate records) under network retry conditions.
- Don't assume status code correctness from documentation alone — APIs frequently drift from their documented contract as they evolve; verifying actual behavior against real requests is exactly what API tests are for.
- When an API returns a 5xx error during testing, treat it as a genuine defect worth investigating immediately, not just a test failure to route around — a 5xx specifically indicates the *server* failed, which is categorically different from a 4xx (the client sent something invalid).

## Common Pitfalls

- Confusing PUT and PATCH — using PUT to send a partial update (missing fields may get wiped out, since PUT means "replace entirely") when PATCH was the correct, safer choice.
- Assuming idempotency without testing it — a common gap, since idempotency bugs often don't show up until real-world retry conditions (network blips, client-side retry logic) trigger them in production.
- Treating 401 and 403 as interchangeable "auth failed" cases in test assertions — they represent meaningfully different failure modes (not authenticated vs. not authorized) and should be tested and asserted on distinctly.
- Not testing the specific 4xx code returned for invalid input — asserting only "status is not 200" instead of the specific expected code (400 vs. 422, for example) misses real API contract regressions.

## Interview Notes

- Be ready to state which HTTP methods are idempotent and safe, and explain both terms precisely, with an example of why idempotency matters practically (retry safety).
- Know the 401 vs. 403 distinction cold — a very commonly asked, easy-to-get-wrong question.
- Be able to describe how you'd actually test idempotency (not just define it) — calling the same request twice and comparing resulting state, not just checking both responses individually succeeded.

## References

- [MDN — HTTP Request Methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
- [MDN — HTTP Response Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)