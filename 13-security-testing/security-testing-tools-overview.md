# Security Testing Tools Overview

## What It Is

This note covers the two most widely used dedicated web security testing tools — **OWASP ZAP** (free, open-source) and **Burp Suite** (industry-standard, with a free Community edition and paid Professional tier) — at the level an SDET needs: what they do, how to run a basic automated scan, and where their output fits into the "escalate to specialist" boundary established in [Security Testing Fundamentals](./security-testing-fundamentals.md).

## Why It Matters

- These tools automate much of what earlier notes in this folder covered manually (injection probing, XSS payload testing, header checking) — running an automated scan is a practical way to get broad, systematic coverage quickly, complementing the targeted manual/scripted tests from earlier notes.
- Being able to name and describe basic usage of these specific, industry-standard tools is a common, concrete interview signal — "have you used a security scanning tool" is frequently asked, and a specific, accurate answer is meaningfully stronger than "I understand security concepts."
- Understanding what these tools' automated scans *do and don't* catch reinforces the specialist-escalation boundary from [Security Testing Fundamentals](./security-testing-fundamentals.md) — an automated scan is a useful first pass, not a substitute for expert manual penetration testing.

## How It Works

**OWASP ZAP (Zed Attack Proxy):**
- Free, open-source, maintained under the OWASP umbrella.
- Runs as a proxy between your browser and the target application, or can perform automated spidering/scanning of a target URL directly.
- Includes automated scanners for many OWASP Top 10 categories (XSS, SQL injection probing, missing security headers) and can generate a structured findings report.
- Has a CLI/Docker mode suitable for CI integration (a "baseline scan" for quick checks, a fuller "full scan" for deeper, slower analysis).

**Burp Suite:**
- Industry-standard, widely used by dedicated security professionals; Community edition is free but with reduced automation (no automated scanner); Professional edition (paid) includes the full automated scanner.
- Functions primarily as an intercepting proxy — letting you see and manipulate every request/response between browser and application, which is valuable for manual, exploratory security testing beyond what pure automation catches.
- Widely referenced in security testing job postings and penetration testing methodology — familiarity with its basic interception workflow is a recognized, practical skill.

**What automated scanning catches well vs. not:** these tools are strong at systematically probing for well-known vulnerability patterns (missing headers, common XSS/injection payloads, outdated software fingerprinting) — but they don't understand business logic, so they can't catch a logic flaw (e.g., a discount code that can be applied twice) the way a human tester or a targeted manual test would.

## Example

**Running an OWASP ZAP baseline scan via Docker — a lightweight, CI-friendly automated pass:**
```bash
docker run -v $(pwd):/zap/wrk/:rw \
  -t zaproxy/zap-stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html

# Produces an HTML report flagging findings like:
#   - Missing Anti-clickjacking Header
#   - X-Content-Type-Options Header Missing
#   - Cookie No HttpOnly Flag
# ... many of which overlap directly with checks from
# Security Headers & Configuration Testing, now automated
```

**Integrating a ZAP baseline scan into CI as a non-blocking, informational check (given automated scanners can produce false positives worth human review before treating as blocking):**
```yaml
# .github/workflows/zap-scan.yml
name: OWASP ZAP Baseline Scan

on:
  schedule:
    - cron: '0 3 * * 1'   # weekly, not on every PR — this scan is slower
                            # than typical CI checks

jobs:
  zap_scan:
    runs-on: ubuntu-latest
    steps:
      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'https://staging.example.com'
          # Results posted as an issue/artifact for human review,
          # not an automatic blocking gate — see the false-positive
          # caution in Production Considerations
```

**Using Burp Suite's proxy for manual, exploratory security testing** (illustrative workflow, not something scripted into CI):
```text
1. Configure browser to route traffic through Burp's local proxy
2. Browse the application normally, performing a typical user flow
   (login, add to cart, checkout)
3. Burp's "Proxy > HTTP History" tab captures every request/response —
   inspect these directly for issues automated tools might miss
   (a discount calculation exposed in a response that shouldn't be
   client-visible, an authorization check that only happens client-side)
4. Use "Repeater" to manually modify and resend a specific request —
   e.g., changing a user ID in an API call to test for BOLA
   (see API Security Testing Basics), directly and interactively
```

## Production Considerations

- Run automated scans (ZAP baseline, or similar) on a schedule against a staging environment, not as a blocking gate on every PR — these scans are slower than typical CI checks and can produce false positives that need human review before being treated as confirmed findings.
- Never point these tools at a production environment without explicit authorization — even a "passive" scan can generate significant traffic, and an active scan (which sends actual attack payloads) against production without authorization is both risky and potentially against your organization's policies.
- Treat automated scan findings as a triage starting point (same principle as [AI-Assisted Defect Analysis](../11-ai-testing/ai-assisted-defect-analysis.md)'s clustering) — confirm and prioritize findings before filing them as defects, since automated scanners can produce false positives.

## Common Pitfalls

- Running an active security scan against a production environment without proper authorization — this is a serious operational and potentially legal issue, not just a testing mistake.
- Treating every automated scan finding as a confirmed, must-fix defect without triage — false positives are common in automated scanning, and filing them uncritically erodes trust in the scan's output over time.
- Relying on automated scanning alone and assuming it provides complete security coverage — these tools don't understand business logic and can't catch logic-flaw vulnerabilities a human tester would find through manual exploration.
- Confusing Burp Suite Community (manual-only, no automated scanner) with Professional (includes automated scanning) when discussing capabilities — a common, specific mix-up worth getting right.

## Interview Notes

- Be ready to name OWASP ZAP and Burp Suite specifically, and describe at least a basic usage workflow for one (running a baseline scan, or using the proxy/Repeater for manual testing) — specific, accurate tool knowledge is a strong, concrete signal.
- Understand what automated scanning catches well (known patterns, missing headers) versus what it misses (business logic flaws) — this reinforces the specialist-escalation judgment from [Security Testing Fundamentals](./security-testing-fundamentals.md).
- Be able to explain why running an active scan against production without authorization is a serious issue, not just a technical footnote — this shows real, practical security judgment.

## References

- [OWASP ZAP — Official Site](https://www.zaproxy.org/)
- [PortSwigger — Burp Suite](https://portswigger.net/burp)