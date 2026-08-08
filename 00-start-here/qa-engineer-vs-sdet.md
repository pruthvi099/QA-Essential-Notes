# QA Engineer vs. SDET

## What It Is

**QA Engineer** and **SDET (Software Development Engineer in Test)** are related but distinct roles in quality engineering. The lines blur across companies, but the core distinction is about *what tools you use to ensure quality*:

- A **QA Engineer** focuses on testing the product — designing test cases, executing them (manually and/or with automation tools), and reporting defects. The primary skill is testing methodology.
- An **SDET** focuses on *building the systems that enable testing at scale* — writing production-quality code for test frameworks, CI/CD integration, tooling, and infrastructure, in addition to designing tests. The primary skill set overlaps heavily with software engineering.

Neither title is standardized industry-wide — some companies use "SDET" for what is functionally a QA Automation Engineer role, and vice versa. What matters is understanding the underlying skill spectrum.

## Why It Matters

- Recruiters and interviewers use this distinction to calibrate expectations: an SDET interview typically includes data structures/algorithms and system design, not just testing theory — because you're expected to build tools, not just use them.
- Knowing where you sit on this spectrum (and where you want to grow) shapes what you should be learning next — this repo itself is structured around the QA → QA Automation → SDET → Advanced SDET progression for that reason.
- Career growth in this field usually flows toward SDET/QA Architect responsibilities — understanding the distinction early helps you build the right skills instead of only accumulating tool familiarity.

## How It Works

| Aspect | QA Engineer | SDET |
|---|---|---|
| Primary focus | Testing the product | Building tools/frameworks that enable testing |
| Coding skill | Basic to moderate (scripts, simple automation) | Strong (production-quality code, design patterns) |
| Typical work | Test case design, manual + automated execution, defect reporting | Framework architecture, CI/CD pipelines, custom tooling, mentoring on automation |
| Ownership | Test coverage for a feature/product area | Test infrastructure used across the team/org |
| Example deliverable | A suite of Playwright tests for the checkout flow | The Playwright framework itself — fixtures, reporting, CI integration, parallel execution strategy — that other testers build tests on top of |

A useful mental model: a QA Automation Engineer *uses* a framework to write tests; an SDET *builds and maintains* that framework, in addition to writing tests.

## Example

The same need — "the team needs a way to run API tests with authentication handled automatically" — approached at each level:

**QA Engineer approach:** writes individual test scripts that each manually fetch a token and pass it in headers.

```python
import requests

def get_token():
    resp = requests.post("https://api.example.com/auth", json={
        "username": "test_user", "password": "Pass@123"
    })
    return resp.json()["token"]

def test_get_orders():
    token = get_token()
    resp = requests.get(
        "https://api.example.com/orders",
        headers={"Authorization": f"Bearer {token}"}
    )
    assert resp.status_code == 200
```

**SDET approach:** builds a reusable fixture so every test in the suite gets an authenticated client automatically, with token caching and config-driven environments.

```python
import pytest
import requests

@pytest.fixture(scope="session")
def api_client():
    token = requests.post(f"{BASE_URL}/auth", json=CREDENTIALS).json()["token"]
    session = requests.Session()
    session.headers.update({"Authorization": f"Bearer {token}"})
    session.base_url = BASE_URL
    yield session
    session.close()

def test_get_orders(api_client):
    resp = api_client.get(f"{api_client.base_url}/orders")
    assert resp.status_code == 200
```

The SDET version is reusable across the entire test suite, config-driven, and handles setup/teardown — this is the kind of infrastructure thinking that separates the two roles.

## Production Considerations

- Growing from QA Engineer to SDET is primarily a software engineering skill investment: version control workflows, code review practices, design patterns (Page Object Model, fixtures, factories), and CI/CD — not just learning more testing theory.
- Teams need both roles/skillsets — pure SDET teams without strong manual/exploratory testing instinct miss usability and edge-case bugs; pure QA teams without SDET-level tooling can't scale automation across a growing codebase.
-