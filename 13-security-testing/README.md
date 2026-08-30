# 13 — Security Testing

Security testing an SDET can reasonably own — OWASP Top 10 fundamentals, injection, XSS, authentication/session security, CSRF/clickjacking, headers, dependency scanning, and basic tooling — with a clear line to where specialist penetration testing takes over. Read [04-api-testing](../04-api-testing/) first for the API security basics this folder builds on.

## Notes

1. [Security Testing Fundamentals](./security-testing-fundamentals.md) — OWASP Top 10, and SDET scope vs. specialist scope
2. [Injection Attacks & Testing](./injection-attacks-and-testing.md) — SQL and command injection, mechanism and defensive testing
3. [XSS Testing](./xss-testing.md) — Reflected, stored, and DOM-based cross-site scripting
4. [Authentication & Session Security Testing](./authentication-and-session-security-testing.md) — Session fixation, password policy, credential stuffing
5. [CSRF & Clickjacking Testing](./csrf-and-clickjacking-testing.md) — Attacks that exploit user trust, not input handling
6. [Security Headers & Configuration Testing](./security-headers-and-configuration-testing.md) — CSP, HSTS, cookie attributes, and misconfiguration checks
7. [Dependency & Supply Chain Security Testing](./dependency-and-supply-chain-security-testing.md) — npm audit, pip-audit, Dependabot
8. [Security Testing Tools Overview](./security-testing-tools-overview.md) — OWASP ZAP and Burp Suite basics

## Related

- [04 — API Testing](../04-api-testing/) — API-specific security testing (BOLA, JWT, OAuth2) this folder extends
- [07 — CI/CD](../07-ci-cd/) — where header checks and dependency scans become automated quality gates