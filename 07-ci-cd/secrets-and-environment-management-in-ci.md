# Secrets & Environment Management in CI

## What It Is

This note covers how credentials (API keys, database passwords, service tokens) and environment-specific configuration (base URLs, feature flags) are securely managed in CI/CD pipelines — extending the "never hardcode secrets" principle mentioned throughout this repo (see [API Authentication](../04-api-testing/api-authentication.md), [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md)) into the concrete mechanics of CI secret stores and multi-environment pipeline configuration.

## Why It Matters

- A leaked credential in CI is a genuinely severe security incident, not just a testing hygiene issue — CI logs, workflow files, and artifact outputs are frequently more widely accessible (to a whole team, sometimes publicly for open-source repos) than a local development environment, making CI a common real-world source of credential leaks.
- Running the same test suite against different environments (local, staging, production-adjacent) safely requires deliberate environment-driven configuration in the pipeline itself — a hardcoded value anywhere in this chain breaks that flexibility and creates real risk (e.g., a staging suite accidentally pointed at production).
- Interviewers ask about CI secrets management specifically because it's an area where a wrong answer ("I just put the API key in the script") signals a real security gap in practical experience, not just a theoretical knowledge gap.

## How It Works

**Core principles, platform-agnostic:**

1. **Never commit secrets to version control** — not even in a `Jenkinsfile` or workflow YAML, since these are typically committed alongside the application code.
2. **Use the CI platform's built-in secret store** — GitHub Actions Secrets, Jenkins Credentials — which encrypts values and masks them in logs automatically.
3. **Inject secrets as environment variables at runtime**, never as literal values baked into a script or config file.
4. **Scope secrets narrowly** — a secret needed only for a deploy step shouldn't be exposed to every job/step in the pipeline.
5. **Rotate secrets periodically**, and immediately if a leak is ever suspected — treat this the same as any production credential incident response.

**Multi-environment configuration** — combine secrets (sensitive values) with non-sensitive environment-specific config (base URLs, feature flags) driven by which environment the pipeline stage targets, following the same typed-config pattern from [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md).

## Example

**GitHub Actions — secrets and multi-environment configuration combined:**

```yaml
jobs:
  test_staging:
    runs-on: ubuntu-latest
    environment: staging   # GitHub Environments — scopes secrets to this specific environment
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Run tests against staging
        run: npx playwright test
        env:
          TEST_ENV: staging
          BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          API_KEY: ${{ secrets.STAGING_API_KEY }}
          # Secrets are automatically masked in logs — even an accidental
          # `echo $API_KEY` would show *** instead of the real value

  test_production_smoke:
    runs-on: ubuntu-latest
    environment: production   # a DIFFERENT scope — production secrets are
                                # only accessible to jobs explicitly targeting it
    needs: test_staging        # only runs after staging tests pass
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Run smoke tests against production
        run: npx playwright test --grep @smoke
        env:
          TEST_ENV: production
          BASE_URL: ${{ secrets.PRODUCTION_BASE_URL }}
          API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
```

**Jenkins — the equivalent using scoped credential bindings:**

```groovy
pipeline {
    agent any
    stages {
        stage('Test Staging') {
            environment {
                BASE_URL = credentials('staging-base-url')
                API_KEY = credentials('staging-api-key')
            }
            steps {
                sh 'TEST_ENV=staging pytest'
            }
        }
        stage('Smoke Test Production') {
            when { branch 'main' }
            environment {
                BASE_URL = credentials('production-base-url')
                API_KEY = credentials('production-api-key')
            }
            steps {
                sh 'TEST_ENV=production pytest -m smoke'
            }
        }
    }
}
```

A deliberate safeguard against pointing the wrong test tier at production, given the real risk of a misconfigured pipeline:

```yaml
- name: Verify environment safety before production run
  run: |
    if [ "$TEST_ENV" = "production" ] && [ "${{ github.event_name }}" != "schedule" ]; then
      echo "Refusing to run non-scheduled tests against production"
      exit 1
    fi
```

## Production Considerations

- Use the CI platform's environment-scoping features (GitHub Environments, Jenkins folder-level credentials) to ensure production secrets are genuinely inaccessible to jobs/pipelines that shouldn't have them — this is a real access-control boundary, not just an organizational convention.
- Require additional protection (manual approval, restricted branch access) specifically for any job with access to production secrets — the blast radius of a compromised production credential is categorically worse than a staging one.
- Periodically audit which secrets exist and which jobs/pipelines can access them — over time, overly broad secret scoping tends to accumulate as pipelines are copied/extended without re-evaluating access needs.

## Common Pitfalls

- Committing a secret directly into a workflow YAML or `Jenkinsfile`, even temporarily "just for testing" — CI configuration files are typically committed to the repository and this is a genuine, common leak vector.
- Scoping a secret broadly (accessible to every job) when only one specific job actually needs it, unnecessarily widening the blast radius if any part of the pipeline is compromised.
- Not distinguishing sensitive values (API keys, passwords — belong in a secret store) from non-sensitive environment config (a public base URL — fine as a regular variable) and treating everything the same way, either over-protecting harmless config or under-protecting real secrets.
- Forgetting that CI logs can inadvertently expose secret values through debug output or error messages if a secret isn't properly referenced through the platform's masking mechanism — always use the platform's secret injection method, never manually echo or print credential values.

## Interview Notes

- Be ready to explain how you'd manage credentials differently for a staging pipeline versus a production-adjacent pipeline — the environment-scoping concept is a common, practical follow-up question.
- Understand why secrets should never be committed to a `Jenkinsfile`/workflow YAML even though those files "look like configuration" — a common, easy-to-miss distinction for those newer to CI/CD security.
- Be able to describe a real safeguard against accidentally running the wrong test tier against production (as shown in the example) — this demonstrates practical, defensive pipeline design thinking.

## References

- [GitHub Actions — Using Secrets in GitHub Actions](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)
- [Jenkins — Using Credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- [OWASP — CI/CD Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html)