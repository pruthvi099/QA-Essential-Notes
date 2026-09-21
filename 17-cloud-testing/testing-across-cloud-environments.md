# Testing Across Cloud Environments

## What It Is

This note covers managing test execution across multiple cloud-hosted environments — dev, staging, production-adjacent — including environment provisioning strategy (persistent shared environments vs. ephemeral per-PR environments) and the configuration management this requires. This extends [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md)'s layered config specifically to cloud infrastructure, where environment provisioning itself (not just configuration values) becomes a testing concern.

## Why It Matters

- Cloud infrastructure makes a genuinely new environment strategy possible that on-prem infrastructure couldn't easily support: **ephemeral per-PR environments** — a full, isolated copy of the application stack spun up for a single pull request and torn down after — this fundamentally changes how environment-related test flakiness (shared staging environment contention) can be addressed.
- Managing configuration correctly across multiple cloud environments (different account IDs, different resource identifiers, different regions) is a genuine, everyday source of misconfiguration risk if not handled with the same discipline as [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md)'s precedence-order approach.
- This is a practical, current topic — many teams are actively adopting ephemeral environment strategies specifically to solve the shared-staging-environment contention problems covered generally in [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) and [API Test Data Management](../04-api-testing/api-test-data-management.md).

## How It Works

**Persistent shared environments (traditional approach):** one long-lived staging environment shared by the whole team, always running. Simple to reason about, but prone to the contention/data-isolation issues covered in [Manual Test Data Management](../01-manual-testing/manual-test-data-management.md) — multiple engineers' work colliding in the same shared space.

**Ephemeral per-PR environments (cloud-native approach):** a full, isolated environment spun up automatically for each pull request (via infrastructure-as-code tooling), used for that PR's testing, and automatically torn down once the PR merges or closes. Solves contention entirely — each PR gets a genuinely isolated environment — but requires more sophisticated CI/infrastructure tooling and has real, metered provisioning cost per environment.

**Configuration across environments:** extends the layered precedence pattern from [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md) — cloud-specific values (account IDs, region, resource ARNs/identifiers) need the same explicit, fail-loudly-on-invalid-environment discipline, now with cloud-specific values added to the configuration layers.

## Example

**A GitHub Actions workflow provisioning an ephemeral per-PR environment, then tearing it down automatically:**
```yaml
# .github/workflows/pr-environment.yml
name: Ephemeral PR Environment

on:
  pull_request:
    types: [opened, synchronize, closed]

jobs:
  provision:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Provision environment via Terraform
        run: |
          terraform workspace new pr-${{ github.event.number }} || \
            terraform workspace select pr-${{ github.event.number }}
          terraform apply -auto-approve -var="pr_number=${{ github.event.number }}"

      - name: Run E2E tests against the fresh environment
        run: |
          export BASE_URL=$(terraform output -raw app_url)
          npx playwright test

  teardown:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Tear down the PR's environment
        run: |
          terraform workspace select pr-${{ github.event.number }}
          terraform destroy -auto-approve
          terraform workspace select default
          terraform workspace delete pr-${{ github.event.number }}
```

**Extending the layered configuration pattern from [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md) with cloud-specific environment values:**
```typescript
// config/cloud-environments.ts
interface CloudEnvConfig {
  awsAccountId: string;
  region: string;
  baseURL: string;
}

const cloudEnvironments: Record<string, CloudEnvConfig> = {
  dev: {
    awsAccountId: '111111111111',
    region: 'us-east-1',
    baseURL: 'https://dev.example.com',
  },
  staging: {
    awsAccountId: '222222222222',
    region: 'us-east-1',
    baseURL: 'https://staging.example.com',
  },
  // A per-PR environment's config is generated dynamically at
  // provisioning time (as in the Terraform output above), not
  // hardcoded here, since its URL/identifiers don't exist until
  // the environment is actually created
};

export function getCloudConfig(): CloudEnvConfig {
  const env = process.env.TEST_ENV;
  if (env === 'pr') {
    // Dynamically provisioned — read from environment variable
    // set by the CI provisioning step, not from the static config
    return {
      awsAccountId: process.env.PR_AWS_ACCOUNT_ID!,
      region: process.env.PR_REGION!,
      baseURL: process.env.PR_BASE_URL!,
    };
  }
  const config = cloudEnvironments[env as string];
  if (!config) throw new Error(`Unknown cloud environment: ${env}`);
  return config;
}
```

## Production Considerations

- Ephemeral per-PR environments genuinely solve the shared-staging contention problem, but introduce their own cost and complexity — provisioning/teardown time adds to CI runtime, and the infrastructure-as-code tooling itself (Terraform, Pulumi, CDK) needs its own maintenance; this trade-off is worth evaluating against actual team pain points, not adopted reflexively.
- Always verify teardown actually runs, including for PRs that are closed without merging — a forgotten teardown step (or one that only triggers on merge, not on close) leaves orphaned, cost-accumulating environments behind, connecting directly to the resource-cleanup discipline from [Test Data & Volumes in Docker](../09-docker/test-data-and-volumes-in-docker.md).
- For teams not ready for full ephemeral-per-PR infrastructure, a middle ground — a small pool of reusable, periodically-reset staging environments — captures some isolation benefit without the full infrastructure-as-code investment.

## Common Pitfalls

- Adopting ephemeral per-PR environments without accounting for the real provisioning/teardown time added to every PR's CI feedback loop — this needs to be weighed against [Pull Request Test Triggers](../07-ci-cd/pull-request-test-triggers.md)'s fast-feedback goals, not adopted without considering the trade-off.
- Not reliably tearing down ephemeral environments (especially for closed-without-merge PRs), leading to orphaned infrastructure and unexpected cost accumulation.
- Hardcoding cloud-specific configuration values (account IDs, regions) directly in test code instead of the layered configuration approach, making the same mistakes [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md) warns against, now with cloud-specific consequences (accidentally running tests against the wrong AWS account).
- Choosing ephemeral-per-PR environments as a default without evaluating whether the team's actual pain point (contention, flakiness) justifies the added infrastructure complexity and cost.

## Interview Notes

- Be ready to compare persistent shared staging environments versus ephemeral per-PR environments — the isolation benefit versus the added infrastructure complexity and cost trade-off.
- Understand how cloud-specific configuration (account IDs, regions, dynamically-provisioned URLs) extends the layered configuration precedence pattern from [Framework Configuration Architecture](../10-test-framework-design/framework-configuration-architecture.md).
- Be able to describe a realistic CI workflow for provisioning and tearing down an ephemeral test environment, including why reliable teardown matters specifically (cost, not just tidiness).

## References

- [Terraform — Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)
- [AWS — Best Practices for Ephemeral Environments](https://aws.amazon.com/blogs/devops/)