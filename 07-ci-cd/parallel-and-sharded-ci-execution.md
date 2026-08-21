# Parallel & Sharded CI Execution

## What It Is

This note extends [Parallel Execution](../02-automation-python-playwright/parallel-execution.md)'s worker-based parallelism (multiple processes on one machine) to the CI-platform level: **sharding** — splitting a test suite across multiple separate CI runner machines running simultaneously — plus the practical mechanics of configuring this in GitHub Actions and Jenkins specifically.

## Why It Matters

- Worker-based parallelism (multiple processes on one machine) hits a hard ceiling at that machine's CPU/memory limits — sharding across multiple CI runners is the next scaling step once a suite has grown too large for a single machine's parallelism to keep runtime acceptable.
- Sharding strategy directly affects CI cost and runtime — poorly balanced shards (one shard taking 20 minutes while others finish in 5) waste the exact parallelism benefit sharding is meant to provide.
- This is a practical, scale-driven topic — smaller suites rarely need it, but interviewers ask about it specifically for roles at companies with large test suites, to gauge experience with CI performance at real scale rather than just small-project setups.

## How It Works

**The core mechanism:** the test suite is split into N groups ("shards"), each shard runs on its own CI runner in parallel, and results are combined/reported together afterward.

**Shard balancing strategies:**
- **By test count** — simplest: divide the total number of tests evenly across shards (Playwright Test's built-in `--shard` does this).
- **By historical duration** — smarter: balance shards by each test's typical execution time (from previous run data), since not all tests take the same time — this produces more evenly-finishing shards than a simple count-based split.

## Example

**Playwright Test — built-in sharding support (TypeScript):**
```yaml
# .github/workflows/sharded-e2e.yml
name: Sharded E2E Tests
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]   # 4 shards running in parallel, on 4 separate runners
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx playwright install --with-deps

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: npx playwright test --shard=${{ matrix.shard }}/4

      - name: Upload blob report for this shard
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report/
          retention-days: 1

  merge_reports:
    needs: test
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true
      - run: npx playwright merge-reports --reporter html ./all-blob-reports
      - uses: actions/upload-artifact@v4
        with:
          name: merged-html-report
          path: playwright-report/
```

**Python (pytest) — sharding via `pytest-split`, which balances by historical test duration rather than raw count:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt pytest-split --break-system-packages

      - name: Run tests (shard ${{ matrix.shard }}/4, duration-balanced)
        run: |
          pytest --splits 4 --group ${{ matrix.shard }} \
            --splitting-algorithm least_duration \
            --store-durations   # updates the duration cache for future balancing
```

## Production Considerations

- Sharding only pays off once suite runtime has genuinely grown large enough that a single machine's worker-based parallelism (see [Parallel Execution](../02-automation-python-playwright/parallel-execution.md)) is the actual bottleneck — introducing shard-level CI complexity for a suite that already runs in a few minutes adds coordination overhead without meaningful benefit.
- Duration-based shard balancing (as with `pytest-split --splitting-algorithm least_duration`) meaningfully outperforms simple count-based splitting once test durations vary significantly — a shard with several slow E2E tests versus one with many fast unit-style checks will otherwise finish very unevenly.
- Merging reports across shards (as shown in the Playwright example's `merge_reports` job) is necessary to get one unified, readable result — without it, developers have to check N separate shard reports individually, undermining the debugging value covered in [Test Artifacts & Reports in CI](./test-artifacts-and-reports-in-ci.md).

## Common Pitfalls

- Sharding by simple test count without considering duration variance, resulting in unevenly finishing shards where total pipeline time is still bottlenecked by the single slowest shard.
- Not merging shard reports into a single unified view, forcing developers to manually check multiple separate report artifacts to understand overall suite health.
- Introducing sharding prematurely, before it's actually needed — adding CI complexity (multiple jobs, report merging, shard coordination) for a suite that doesn't yet have a real runtime problem.
- Confusing sharding (multiple separate CI runner machines) with worker-based parallelism (multiple processes on one machine) — they solve the same general problem (speed) but operate at different scaling layers, and interviewers often probe this distinction specifically.

## Interview Notes

- Be ready to explain the difference between worker-based parallelism (one machine, multiple processes) and sharding (multiple machines) — a common, specific distinction interviewers check for.
- Understand why duration-based shard balancing outperforms simple count-based splitting, with a concrete example of when the difference matters.
- Be able to describe how you'd decide a suite has grown large enough to justify introducing sharding, rather than defaulting to it from the start — shows judgment about when added complexity is actually worth it.

## References

- [Playwright — Sharding (Node.js)](https://playwright.dev/docs/test-sharding)
- [pytest-split — Documentation](https://jerry-git.github.io/pytest-split/)