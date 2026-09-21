# Cloud Testing Fundamentals

## What It Is

Cloud testing covers what changes when an application runs on cloud infrastructure (AWS, Azure, GCP) rather than traditional on-premises servers — dynamic, ephemeral infrastructure, managed services replacing self-hosted ones, and new failure modes (regional outages, auto-scaling behavior) that don't exist in a fixed, on-prem environment. This note establishes the foundational shift this folder builds on.

## Why It Matters

- Most modern applications run on cloud infrastructure by default now — understanding what's genuinely different about testing in this context (not just "it's on someone else's server") is practical, near-universal knowledge for a current SDET role.
- Cloud infrastructure introduces genuinely new testing concerns — infrastructure that scales dynamically, services that can fail independently, environments that are provisioned and torn down on demand — none of which map cleanly onto testing assumptions built for a fixed, always-on on-prem server.
- This connects directly to [Docker Compose for Test Stacks](../09-docker/docker-compose-for-test-stacks.md) and [Test Data & Volumes in Docker](../09-docker/test-data-and-volumes-in-docker.md) — cloud testing often builds on the same containerization principles, just at a larger, managed-infrastructure scale.

## How It Works

**What genuinely changes when testing cloud-native/cloud-hosted applications:**

1. **Ephemeral infrastructure** — test environments can be provisioned on-demand and torn down after use (see [Managing Test Code in Monorepos](../08-git-github/managing-test-code-in-monorepos.md)'s CI-scale thinking, now applied to entire environments, not just test selection).
2. **Managed services replace self-hosted ones** — a managed database (RDS, Cloud SQL), managed message queue (SQS, Pub/Sub), or managed cache (ElastiCache) behaves differently under test than a self-hosted equivalent — you can't always freely restart/reset it the way you could a local Docker container (see [Docker Compose for Test Stacks](../09-docker/docker-compose-for-test-stacks.md)).
3. **New failure modes** — regional outages, auto-scaling lag (a burst of traffic before new instances spin up), and inter-service network latency between cloud regions are genuine, testable failure categories that don't exist for a single fixed on-prem server.
4. **Shared responsibility for infrastructure reliability** — the cloud provider guarantees certain infrastructure-level reliability (a managed database's own uptime), but application-level resilience to *that* infrastructure's occasional hiccups (a brief connection drop, a throttled API call) is still the application team's responsibility to test for.

## Example

A comparison table illustrating the concrete testing implications of moving from on-prem to cloud-native, grounding the abstract distinction in specific examples:

```text
On-Prem Assumption                    Cloud Reality & Testing Implication

"The database is always the same      A managed database can have brief
 fixed server, always running"        connectivity blips, maintenance
                                       windows, and read-replica lag —
                                       worth testing resilience to these
                                       (see Testing with Cloud Storage
                                       and Managed Services)

"Traffic capacity is fixed —          Auto-scaling means capacity
 if we exceed it, we know exactly     changes dynamically; a traffic
 what happens"                        spike может briefly outpace new
                                       instance provisioning — worth
                                       testing this scaling LAG
                                       specifically (see Multi-Region
                                       and Scalability Testing)

"Our test environment is a fixed,     Test environments can be
 always-on server we SSH into"        ephemeral — spun up per test run,
                                       torn down after — requiring
                                       different setup/teardown thinking
                                       (see Testing Across Cloud
                                       Environments)

"If it's down, it's ALL down"         A specific cloud REGION can have
                                       an outage while others remain
                                       healthy — worth testing whether
                                       the application handles regional
                                       failover correctly, if it's
                                       architected for multi-region
```

## Production Considerations

- Cloud testing doesn't replace the testing principles covered throughout this repo — it adds new categories of risk (infrastructure ephemerality, managed-service behavior, regional failure) on top of the same functional/API/performance testing fundamentals, not instead of them.
- Coordinate with DevOps/platform engineering on what infrastructure-level reliability is *already* guaranteed by the cloud provider/platform team versus what genuinely needs application-level resilience testing — this avoids either redundant testing of provider-guaranteed reliability or, worse, missing genuine application-level gaps.
- Cloud testing costs money differently than on-prem testing — provisioning ephemeral test environments, running tests against managed services, and multi-region testing all have real, metered cost implications worth understanding (see [Cloud Cost Awareness for Test Environments](./cloud-cost-awareness-for-test-environments.md)).

## Common Pitfalls

- Assuming cloud infrastructure behaves identically to on-prem infrastructure for testing purposes, missing genuinely new failure modes (auto-scaling lag, managed-service connectivity blips, regional outages) that simply don't exist in a fixed on-prem context.
- Testing only against a fully-provisioned, stable cloud environment and never testing the *transition* states (environment spinning up, auto-scaling in progress) that are unique to dynamic cloud infrastructure.
- Not distinguishing what the cloud provider already guarantees (infrastructure uptime SLAs) from what the application team still needs to test for (graceful degradation when that infrastructure has a brief, provider-acknowledged hiccup).
- Treating "cloud testing" as an entirely separate discipline from the functional/API/performance testing covered elsewhere in this repo, rather than recognizing it as an additional layer of considerations on top of those same fundamentals.

## Interview Notes

- Be ready to explain concretely what changes when testing a cloud-native application versus a traditional on-prem one — ephemeral infrastructure, managed services, new failure modes — with a specific example of each.
- Understand the shared-responsibility framing: the cloud provider guarantees certain infrastructure reliability, but application-level resilience to that infrastructure's normal, occasional hiccups is still a testing responsibility.
- Be able to describe how cloud testing builds on (rather than replaces) the functional/API/performance testing fundamentals covered elsewhere in this repo.

## References

- [AWS — Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/)
- [Google Cloud — Site Reliability Engineering](https://sre.google/)