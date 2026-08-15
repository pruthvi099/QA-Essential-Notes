# 04 — API Testing

REST API testing fundamentals through advanced practice — validation, auth, security, contracts, and workflow testing. Read [02-automation-python-playwright](../02-automation-python-playwright/) first for the base `APIRequestContext` tooling used throughout these notes.

## Notes

1. [HTTP Methods & Status Codes](./http-methods-and-status-codes.md) — Method semantics, idempotency, and status code categories
2. [Request & Response Validation](./request-response-validation.md) — The five layers of a thorough API assertion
3. [JSON Schema Validation](./json-schema-validation.md) — Declarative structural/type validation at scale
4. [API Authentication](./api-authentication.md) — API keys, Basic Auth, and session-based auth
5. [JWT Testing](./jwt-testing.md) — Token structure, claims, and expired/tampered/malformed token handling
6. [OAuth2 Testing](./oauth2-testing.md) — Authorization Code and Client Credentials flows, scope enforcement
7. [Negative Testing for APIs](./negative-testing-for-apis.md) — Systematically testing invalid input handling
8. [API Chaining & Workflow Testing](./api-chaining-and-workflow-testing.md) — Multi-step, dependent request flows
9. [Idempotency & Rate Limiting](./idempotency-and-rate-limiting.md) — Idempotency keys and throttling behavior
10. [API Contract Testing](./api-contract-testing.md) — Consumer-driven contracts vs. schema validation
11. [API Test Data Management](./api-test-data-management.md) — Isolated, API-driven setup and teardown
12. [API Security Testing Basics](./api-security-testing-basics.md) — BOLA, data exposure, mass assignment, and more
13. [Database Validation from API Tests](./database-validation-from-api-tests.md) — Verifying real persistence beyond the response

## Related

- [02 — Automation (Python + Playwright)](../02-automation-python-playwright/) — the `APIRequestContext` foundation used throughout
- [05 — SQL & Database Testing](../05-sql-database-testing/) — deeper database validation techniques
- [13 — Security Testing](../13-security-testing/) — broader security testing beyond the API layer