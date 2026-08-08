# SDLC Models

## What It Is

The Software Development Life Cycle (SDLC) is the sequence of phases a software product goes through — requirements, design, development, testing, deployment, maintenance. An **SDLC model** is a specific way of arranging and timing those phases. The model a team uses directly shapes *when* and *how* testing happens, which is why SDETs need to recognize them, not just developers.

## Why It Matters

- The SDLC model determines whether testing is a late-stage gate (Waterfall) or a continuous, parallel activity (Agile/DevOps) — this changes what your automation strategy needs to look like.
- Interviewers use "what SDLC model have you worked in?" to gauge whether you understand testing as embedded in a process, not a bolt-on phase.
- Knowing the V-Model specifically matters because it explicitly maps each development phase to a corresponding testing phase — it's the conceptual basis for the STLC (see [What Is QA](./what-is-software-quality-assurance.md)).

## How It Works

**Waterfall** — strictly sequential: Requirements → Design → Development → Testing → Deployment → Maintenance. Testing only starts after development is fully complete.

**V-Model** — an extension of Waterfall where each development phase has a corresponding testing phase planned in parallel, even though execution still happens late:

```text
Requirements  ─────────────────  Acceptance Testing
   High-Level Design ────────  System Testing
      Low-Level Design ─────  Integration Testing
         Coding  ───────────  Unit Testing
```

**Agile (Scrum/Kanban)** — iterative and incremental: small chunks of requirements → design → build → test are completed in short cycles (sprints), with testing happening continuously *within* each sprint rather than as a final phase.

**DevOps / Continuous Testing** — testing is fully integrated into the CI/CD pipeline; every code change triggers automated tests automatically, blurring the line between "development" and "testing" phases entirely.

| Model | When testing happens | Automation's role |
|---|---|---|
| Waterfall | After development is complete | Limited — often manual, late-stage |
| V-Model | Planned early, executed late, phase-mapped | Test cases can be designed early; automation still executes late |
| Agile | Continuously, within each sprint | Essential — needed to keep pace with fast iterations |
| DevOps | Continuously, on every commit/PR | Core to the pipeline — tests gate every deployment |

## Example

How the same feature — "add a discount code field to checkout" — flows differently depending on the model:

**Waterfall/V-Model:** Full requirements are signed off, entire application is built, then a