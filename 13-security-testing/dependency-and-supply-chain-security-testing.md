# Dependency & Supply Chain Security Testing

## What It Is

This note covers scanning application dependencies (npm/pip packages, and their own transitive dependencies) for known vulnerabilities, using tools like `npm audit`, `pip-audit`, and automated services like Dependabot/Snyk. This is one of the most fully automatable categories of security testing — the tooling exists specifically to make this a continuous, low-effort CI check rather than manual review work.

## Why It Matters

- Modern applications depend on hundreds or thousands of third-party packages (including transitive dependencies pulled in indirectly) — a vulnerability in any one of them is a vulnerability in the application, even though no one on the team wrote that code directly.
- This category of risk is distinct from application-code vulnerabilities (injection, XSS) covered elsewhere in this folder — it's about the *supply chain* of code the application depends on, which requires different tooling and a different mental model (tracking known-vulnerability databases, not testing application behavior directly).
- Because this is highly automatable, it's genuinely low-effort, high-value security testing — an SDET advocating for and setting up this kind of scanning in CI is a concrete, practical contribution requiring minimal ongoing manual work once configured.

## How It Works

**Vulnerability scanning tools, by ecosystem:**
- **JavaScript/Node** — `npm audit` (built into npm), or more comprehensive third-party tools (Snyk, Socket).
- **Python** — `pip-audit` (checks installed packages against the Python Packaging Advisory Database).
- **GitHub-native** — Dependabot, which automatically scans a repository's dependency manifests and can open pull requests updating vulnerable packages to patched versions.

**What these tools actually check:** each scans installed package versions against a known-vulnerability database (e.g., the National Vulnerability Database, GitHub Advisory Database), flagging packages with published CVEs (Common Vulnerabilities and Exposures) and typically indicating severity and whether a patched version is available.

**Beyond scanning — supply chain risk more broadly:** verifying packages come from trusted registries, being cautious about newly-published or unusually low-download packages, and reviewing what a new dependency actually does before adding it — scanning catches *known* vulnerabilities in existing dependencies, but doesn't protect against a newly-introduced malicious package that hasn't been flagged yet.

## Example

**Running dependency audits locally and in CI:**
```bash
# Node/npm
npm audit

# Example output:
# found 3 vulnerabilities (1 low, 1 moderate, 1 high)
#   high: Prototype Pollution in lodash <4.17.21
#     Fix available via `npm audit fix`

# Python
pip install pip-audit --break-system-packages
pip-audit

# Example output:
# Found 1 known vulnerability in 1 package
#   requests 2.25.0 - GHSA-xxxx-xxxx-xxxx (fixed in 2.31.0)
```

**Integrating as a CI quality gate (extending [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md)):**
```yaml
# .github/workflows/security-scan.yml
name: Dependency Security Scan

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'   # also run weekly, since new CVEs are published continuously

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci

      - name: Run npm audit
        run: |
          npm audit --audit-level=high
          # Fails the build on HIGH or CRITICAL severity findings —
          # a chosen threshold, not "zero tolerance for everything,"
          # since some low-severity findings may have no practical
          # exploit path in this specific application's context
```

**A Dependabot configuration, automating ongoing dependency update PRs:**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

## Production Considerations

- Set a deliberate severity threshold for what blocks a CI build (e.g., high/critical severity only) rather than zero-tolerance for every finding — some low-severity vulnerabilities have no realistic exploit path in a given application's specific usage of that package, and blocking on everything can train teams to routinely override the gate (the same risk pattern discussed in [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md)).
- Run scans on a schedule (weekly, as shown) in addition to on every PR — new CVEs are published continuously against packages that haven't changed in your own codebase, so a scan needs to run periodically even without a new commit triggering it.
- Automated dependency-update PRs (Dependabot) still need human review before merging, especially for major version bumps that could include breaking changes — automating the *detection and PR creation* doesn't mean automating the *decision to merge* blindly.

## Common Pitfalls

- Running dependency scans only manually/occasionally instead of continuously in CI, missing newly published vulnerabilities against unchanged code.
- Setting zero-tolerance blocking on every finding regardless of severity, causing frequent, disruptive build failures for low-risk issues and training the team to bypass the gate.
- Blindly auto-merging dependency update PRs without reviewing for breaking changes, especially on major version bumps.
- Assuming dependency scanning alone covers all supply chain risk — it catches *known* vulnerabilities in existing packages, not a newly introduced malicious package with no CVE history yet, which requires a different kind of scrutiny (reviewing what a new dependency actually does before adopting it).

## Interview Notes

- Be ready to name specific tools for at least two ecosystems (`npm audit`/pip-audit, Dependabot) and describe how you'd integrate one into a CI pipeline.
- Understand why a deliberate severity threshold (not zero-tolerance) is the practical approach for CI gating, connecting back to the general quality-gate calibration principle from [Quality Gates & Build Failures](../07-ci-cd/quality-gates-and-build-failures.md).
- Be able to explain the distinction between scanning for known vulnerabilities in existing dependencies versus the broader, harder problem of a newly introduced malicious package — showing awareness that dependency scanning is necessary but not fully sufficient supply chain protection.

## References

- [npm — npm audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)
- [pip-audit — PyPA](https://pypi.org/project/pip-audit/)
- [GitHub — About Dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates)
- [OWASP — Vulnerable and Outdated Components](https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/)