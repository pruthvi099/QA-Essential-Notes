# Jenkins for Test Automation

## What It Is

Jenkins is a widely used, self-hosted, open-source automation server, commonly used for CI/CD in enterprises with more complex or legacy infrastructure needs than a fully managed platform like GitHub Actions provides. Pipelines are defined via a `Jenkinsfile` — either Declarative Pipeline syntax (structured, easier to read) or Scripted Pipeline syntax (Groovy-based, more flexible but more complex).

## Why It Matters

- Despite newer platforms like GitHub Actions gaining popularity, Jenkins remains extremely common in larger, established enterprises — many SDET roles, especially at larger or more traditional companies, still require Jenkins fluency specifically.
- Jenkins' plugin ecosystem and self-hosted flexibility make it suited to complex enterprise needs (custom infrastructure, on-premises requirements, integration with legacy systems) that fully managed CI platforms don't always accommodate as easily.
- Understanding Jenkins alongside GitHub Actions (see [GitHub Actions for Test Automation](./github-actions-for-test-automation.md)) demonstrates CI/CD platform-agnostic thinking — the underlying testing strategy concepts (staging, artifacts, gates) transfer between platforms even though the syntax differs.

## How It Works

**Core concepts:**
- **Jenkinsfile** — a file (typically committed to the repository) defining the pipeline, supporting Declarative or Scripted syntax.
- **Stage** — a logical grouping of steps (e.g., "Build," "Test," "Deploy"), shown visually in Jenkins' pipeline view.
- **Agent** — specifies where a pipeline/stage runs (a specific node, a Docker container, any available agent).
- **Post section** — actions that run after the pipeline/stage completes, regardless of outcome (`always`), or conditionally (`success`, `failure`) — the Jenkins equivalent of GitHub Actions' `if: always()`.

## Example

A Declarative Pipeline running a Python Playwright suite, structured with staged execution similar to the GitHub Actions example:

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        BASE_URL = credentials('staging-base-url')
        TEST_ENV = 'staging'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt --break-system-packages'
                sh 'playwright install --with-deps chromium'
            }
        }

        stage('Run Smoke Tests') {
            when {
                changeRequest()   // only runs on pull request builds
            }
            steps {
                sh 'pytest -m smoke --junitxml=results/smoke-results.xml'
            }
        }

        stage('Run Full Regression') {
            when {
                branch 'main'   // only runs on merges to main
            }
            steps {
                sh 'pytest -m regression --junitxml=results/regression-results.xml'
            }
        }
    }

    post {
        always {
            // Publishes JUnit-format results to Jenkins' built-in test
            // reporting UI, regardless of whether the build passed or failed
            junit 'results/*.xml'

            archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
        }
        failure {
            // Jenkins' post-failure hook — a natural place to wire up
            // team notifications (Slack, email) for a broken build
            echo 'Build failed — notifying team'
        }
    }
}
```

A parallel stage structure for running a browser matrix, the Jenkins equivalent of GitHub Actions' matrix strategy:

```groovy
stage('Cross-Browser Tests') {
    parallel {
        stage('Chromium') {
            steps { sh 'pytest --browser chromium' }
        }
        stage('Firefox') {
            steps { sh 'pytest --browser firefox' }
        }
        stage('WebKit') {
            steps { sh 'pytest --browser webkit' }
        }
    }
}
```

## Production Considerations

- Use Jenkins' `credentials()` binding (as shown for `BASE_URL` above) rather than hardcoding secrets in the `Jenkinsfile` — Jenkins' built-in credential store integrates with this binding to keep secrets out of version control, the same principle as GitHub Actions Secrets.
- The `post { always {...} }` block is where test reports and artifacts should always be published, regardless of pipeline outcome — mirroring the `if: always()` pattern from GitHub Actions, and just as easy to forget with the same consequence (losing debugging artifacts on failure).
- Jenkins' `when { changeRequest() }` and `when { branch 'main' }` conditionals are how stage-level test tiering (smoke on PR, full regression on merge) is implemented — the same strategic staging decision covered in [Smoke vs. Regression in CI](./smoke-vs-regression-in-ci.md), just expressed in Jenkins' specific syntax.

## Common Pitfalls

- Hardcoding credentials directly in the `Jenkinsfile` instead of using Jenkins' credential binding — since the `Jenkinsfile` is typically committed to the repository, this is a direct secrets leak.
- Forgetting the `post { always {...} }` block for report/artifact publishing, losing exactly the information needed to debug a failure — the same consequence as the GitHub Actions `if: always()` omission.
- Not using `parallel` for independent stages (like a browser matrix) that could run concurrently, leaving unnecessary time on the table by running them sequentially instead.
- Overcomplicating a pipeline with Scripted Pipeline syntax (full Groovy scripting) when Declarative Pipeline syntax would suffice — Declarative is generally easier to read/maintain and sufficient for most standard test automation pipeline needs.

## Interview Notes

- Be ready to sketch a basic Declarative Pipeline `Jenkinsfile` with stages for checkout, install, and test execution — a common practical exercise for roles using Jenkins specifically.
- Understand the Declarative vs. Scripted Pipeline distinction, and be able to explain why Declarative is generally preferred for straightforward test automation pipelines.
- Be able to translate CI/CD concepts (staging, artifacts, secrets, quality gates) between Jenkins and GitHub Actions syntax — this shows platform-agnostic understanding rather than memorized syntax for just one tool.

## References

- [Jenkins — Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins — Using Credentials](https://www.jenkins.io/doc/book/using/using-credentials/)