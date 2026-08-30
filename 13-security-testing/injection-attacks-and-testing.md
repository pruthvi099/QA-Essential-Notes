# Injection Attacks & Testing

## What It Is

Injection attacks occur when untrusted input is passed to an interpreter (a SQL query, a shell command, an LDAP query) in a way that lets the attacker alter its intended structure — causing it to execute unintended commands or access unauthorized data. This note covers SQL injection and command injection specifically, the two most common forms an SDET is likely to encounter, and how to test defensively for them without needing to construct a full working exploit.

## Why It Matters

- Injection has historically been one of the most severe and common web application vulnerability categories on the OWASP Top 10 — a successful injection attack can expose, modify, or delete an entire database's contents, or in the case of command injection, execute arbitrary commands on the server itself.
- This directly extends [Negative Testing for APIs](../04-api-testing/negative-testing-for-apis.md)'s malformed-input testing — many of the same techniques (boundary/negative values) apply here, just with a security lens on top: not just "does the API reject this correctly" but "does it reject this *safely*, without exposing internals."
- Understanding injection mechanically (not just as a buzzword) is a common, specific interview topic — being able to explain *why* a query becomes vulnerable, not just that "SQL injection is bad," signals real understanding.

## How It Works

**SQL injection — the core mechanism:** occurs when user input is concatenated directly into a SQL query string instead of being properly parameterized, letting an attacker's input change the query's actual structure.

```sql
-- Vulnerable pattern (input concatenated directly into the query)
SELECT * FROM users WHERE email = '" + userInput + "'

-- If userInput is:  ' OR '1'='1
-- The resulting query becomes:
SELECT * FROM users WHERE email = '' OR '1'='1'
-- This returns ALL rows, since '1'='1' is always true —
-- a classic authentication-bypass injection pattern
```

**The fix — parameterized queries (prepared statements):** the query structure and the data are sent separately to the database, so user input can never alter the query's structure, no matter what characters it contains.

**Command injection — the core mechanism:** occurs when user input is passed into a system shell command without proper sanitization, letting an attacker append additional commands.

```python
# Vulnerable pattern
os.system(f"ping -c 1 {user_supplied_host}")

# If user_supplied_host is:  example.com; rm -rf /
# The shell executes BOTH commands — the intended ping AND the
# attacker's appended, malicious command
```

## Example

**Testing for SQL injection defensively — verifying the application rejects or safely handles injection-pattern input, without needing to construct a working exploit:**
```python
import pytest

@pytest.mark.parametrize("injection_attempt", [
    "' OR '1'='1",
    "'; DROP TABLE users; --",
    "' UNION SELECT username, password FROM users --",
    "admin'--",
])
def test_login_rejects_sql_injection_patterns(api_context, injection_attempt):
    response = api_context.post("/api/login", data={
        "email": injection_attempt,
        "password": "irrelevant",
    })

    # A SAFE application either rejects this as invalid input (400)
    # or correctly treats it as a non-matching login attempt (401) —
    # it should NEVER succeed, and should NEVER return a 500 with a
    # visible database error/stack trace
    assert response.status in [400, 401]
    assert "sql" not in response.text().lower()
    assert "syntax error" not in response.text().lower()
```

**Testing for command injection in a feature that takes user input used in a system-level operation:**
```python
@pytest.mark.parametrize("injection_attempt", [
    "example.com; rm -rf /",
    "example.com && cat /etc/passwd",
    "example.com | whoami",
    "$(whoami)",
])
def test_network_diagnostic_tool_rejects_command_injection(api_context, injection_attempt):
    response = api_context.post("/api/diagnostics/ping", data={
        "host": injection_attempt,
    })

    # Should reject clearly invalid hostnames, never execute the
    # appended command
    assert response.status == 400
    # Should NOT contain output that would only appear if the
    # injected command actually executed (e.g., contents of /etc/passwd,
    # a whoami result unrelated to a ping operation)
```

**A demonstration of the fix, for context — not something a functional tester writes, but useful to recognize in code review:**
```python
# VULNERABLE — string concatenation
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# SAFE — parameterized query, input can never alter query structure
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

## Production Considerations

- Testing for injection defensively (verifying the app safely rejects malicious-pattern input) is well within functional QA scope; constructing a full, working exploit chain to prove actual data exfiltration is specialist penetration-testing territory (see [Security Testing Fundamentals](./security-testing-fundamentals.md) for this scope distinction).
- A response containing a visible database error message or stack trace in response to injection-pattern input is itself a defect worth flagging immediately, even before determining whether the injection is fully exploitable — information leakage is a real risk on its own (see [Security Headers & Configuration Testing](./security-headers-and-configuration-testing.md)).
- Modern ORMs and query builders parameterize queries by default, significantly reducing (though not eliminating) SQL injection risk — but raw/dynamic query construction anywhere in a codebase reintroduces the risk, making this worth specifically watching for in code review.

## Common Pitfalls

- Only testing with a single, well-known injection string (`' OR '1'='1`) and assuming that's sufficient coverage — different injection patterns (UNION-based, comment-based, boolean-based) can succeed where others fail, so a narrow test set gives false confidence.
- Treating a 500 error with a hidden/generic message as "safe" without checking that no internal details (stack traces, query structure, table names) leak into the response.
- Assuming an application using an ORM is automatically immune to injection — raw/dynamic queries can still exist alongside ORM usage, and it's worth verifying rather than assuming.
- Attempting to actually exploit a suspected injection vulnerability to "prove" it works, beyond the scope of authorized, defensive testing — this crosses into penetration testing territory that requires proper authorization.

## Interview Notes

- Be ready to explain the mechanism of SQL injection precisely — how string concatenation allows an attacker's input to alter query structure — not just that "SQL injection is when someone injects SQL."
- Understand parameterized queries as the actual fix, and be able to explain *why* they work (structure and data sent separately, so data can never become structure).
- Be able to describe how you'd test for injection defensively as a functional tester, and where that scope ends and specialist penetration testing begins.

## References

- [OWASP — Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)