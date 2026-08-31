# Performance Testing with k6

## What It Is

k6 is a modern, open-source, code-first load testing tool where test scripts are written in JavaScript, designed specifically for developer/SDET workflows — version-controlled test scripts, CI-friendly execution, and clear, structured output. This note covers writing and running an actual k6 test, applying the test types and metrics from [Performance Testing Fundamentals](./performance-testing-fundamentals.md) concretely.

## Why It Matters

- k6's code-first approach (JavaScript test scripts, not a GUI-based tool) fits naturally into the same version-control and CI workflows already covered throughout this repo (see [08-git-github](../08-git-github/) and [07-ci-cd](../07-ci-cd/)) — a k6 script reviews, diffs, and runs in CI the same way any other test code does.
- k6 has become a widely adopted, modern standard specifically because it's approachable for engineers already comfortable with JavaScript/TypeScript-based testing (see [03-typescript-playwright](../03-typescript-playwright/)), lowering the barrier for an SDET to pick up performance testing without learning an entirely separate tool's paradigm.
- Being able to write a basic, correct k6 script — not just describe performance testing conceptually — is a genuine, practical skill increasingly expected in SDET roles, given how common load testing has become as a standard part of release readiness.

## How It Works

**Core k6 script structure:**
- **`options`** — configuration defining virtual users (VUs), test duration, and stages (ramp-up/steady/ramp-down).
- **Default exported function** — the code each virtual user executes repeatedly for the test's duration.
- **`check()`** — k6's assertion mechanism, verifying response correctness (status code, body content) without failing the entire test run the way a hard assertion would — checks are tracked as pass/fail metrics.
- **Thresholds** — pass/fail criteria for the whole test run (e.g., "p95 latency must be under 500ms"), which DO cause the test run to fail if violated — distinct from checks, which track individual request-level success.

## Example

**A basic load test simulating realistic checkout traffic, applying the load-test scenario from [Performance Testing Fundamentals](./performance-testing-fundamentals.md):**
```javascript
// checkout-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 200 },   // ramp up to 200 VUs over 2 min
    { duration: '5m', target: 200 },   // stay at 200 VUs for 5 min (steady state)
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],   // 95% of requests must complete under 2s
    http_req_failed: ['rate<0.01'],       // error rate must stay under 1%
  },
};

export default function () {
  const loginRes = http.post('https://staging.example.com/api/login', JSON.stringify({
    email: 'loadtest_user@example.com',
    password: 'Pass@123',
  }), { headers: { 'Content-Type': 'application/json' } });

  check(loginRes, {
    'login succeeded': (r) => r.status === 200,
  });

  const token = loginRes.json('access_token');

  const checkoutRes = http.post('https://staging.example.com/api/orders', JSON.stringify({
    items: [{ product_id: 101, qty: 1 }],
  }), {
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
    },
  });

  check(checkoutRes, {
    'checkout succeeded': (r) => r.status === 201,
  });

  sleep(1);   // pause between iterations, simulating realistic user think-time
}
```

```bash
# Running the test
k6 run checkout-load-test.js

# Output includes a summary showing whether thresholds passed/failed:
#   ✓ http_req_duration..............: p(95)=1.2s
#   ✓ http_req_failed.................: 0.08%
#   ✓ checks...........................: 99.2% ✓ 4960 ✗ 40
```

**A spike test, applying the spike-testing pattern with a different `stages` configuration — showing how k6's same core structure adapts to a different test type:**
```javascript
export const options = {
  stages: [
    { duration: '30s', target: 50 },     // normal baseline traffic
    { duration: '30s', target: 2000 },   // SPIKE — sharp, sudden increase
    { duration: '2m', target: 2000 },    // hold at spike level briefly
    { duration: '1m', target: 50 },      // drop back to baseline — does it recover?
  ],
  thresholds: {
    http_req_failed: ['rate<0.05'],   // slightly more tolerant during a spike
  },
};
```

## Production Considerations

- Use environment variables (`__ENV.BASE_URL`) rather than hardcoded URLs in k6 scripts, following the same environment-configuration discipline from [Enums & Configuration Management](../03-typescript-playwright/enums-and-configuration-management.md) — this lets the same script run against different environments without code changes.
- Run performance tests from infrastructure genuinely close to (or matching) where real users connect from — running a load test from the same data center as the application under test can produce unrealistically low latency numbers that don't reflect real user experience.
- Store k6 scripts in version control alongside the application/test codebase (see [Git Fundamentals for QA](../08-git-github/git-fundamentals-for-qa.md)) — treating performance test scripts with the same code-review discipline as functional test code.

## Common Pitfalls

- Using `check()` alone without `thresholds`, resulting in a test that reports individual failures but never actually fails the overall test run — thresholds are what make a k6 test usable as an actual pass/fail CI gate.
- Not including realistic think-time (`sleep()`) between actions, producing unrealistically aggressive traffic patterns that don't reflect how real users actually interact with the application.
- Hardcoding test data (a single login account) across all virtual users, which can create unrealistic contention or rate-limiting behavior specific to that one account rather than reflecting distributed real usage.
- Running load tests against a differently-provisioned staging environment (fewer resources than production) and drawing conclusions that don't actually transfer to production's real capacity.

## Interview Notes

- Be ready to write a basic k6 script from scratch — VUs, stages, checks, thresholds — a common, practical exercise given k6's current popularity.
- Understand the distinction between `check()` (per-request tracking, doesn't fail the overall run) and `thresholds` (pass/fail criteria for the whole test) — a specific, commonly tested detail.
- Be able to describe how a k6 script's `stages` configuration changes between a load test and a spike test, showing you understand the tool's flexibility across the test types from [Performance Testing Fundamentals](./performance-testing-fundamentals.md).

## References

- [k6 — Official Documentation](https://k6.io/docs/)
- [k6 — Thresholds](https://k6.io/docs/using-k6/thresholds/)