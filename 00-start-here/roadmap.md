# Roadmap

## What It Is

This note is the map for the entire QA-Essential-Notes repository — how the folders are organized, the order they're meant to be read/built in, and what stage of QA/SDET growth each one represents. It's the entry point for a reader who wants to progress from Beginner → QA Engineer → QA Automation Engineer → SDET → Advanced SDET.

## Why It Matters

- A knowledge base without a stated learning path forces every reader to guess where to start — this note removes that ambiguity for both recruiters skimming the repo and engineers using it to actually learn.
- It also acts as a living index: as new folders/notes are added, this is the single place that reflects the intended structure and sequencing, so the repo doesn't drift into an unordered pile of notes.

## How It Works

The repo is organized into 24 numbered folders. The number prefix defines reading/build order, and roughly maps to the four growth stages below.

**Stage 1 — Foundations (Beginner → QA Engineer)**
```text
00-start-here                  Core testing concepts, vocabulary, this roadmap
01-manual-testing               Manual test design, execution, defect reporting
```

**Stage 2 — Automation Fundamentals (QA Engineer → QA Automation Engineer)**
```text
02-automation-python-playwright   Python + Playwright automation
03-typescript-playwright          TypeScript + Playwright automation
04-api-testing                    REST/API testing, auth, schema validation
05-sql-database-testing           Backend/data validation
06-mobile-testing                 Mobile app testing
```

**Stage 3 — Engineering & Delivery (QA Automation Engineer → SDET)**
```text
07-ci-cd                          Pipeline integration, quality gates
08-git-github                     Version control workflows for test code
09-docker                         Containerized test environments
10-test-framework-design          Framework architecture, patterns
11-ai-testing                     AI-assisted testing practices
12-mcp-for-testing                Model Context Protocol for QA tooling
```

**Stage 4 — Advanced SDET Practice**
```text
13-security-testing
14-performance-testing
15-accessibility-visual-testing
16-observability-debugging
17-cloud-testing
18-quality-engineering
19-sdet-interview-preparation
20-real-world-scenarios
21-testing-strategies
22-system-design-for-sdet
23-projects                       Applied, end-to-end projects tying multiple areas together
```

## Example

How this roadmap is meant to be used in practice — a reader new to QA following the repo top to bottom:

```text
1. Read 00-start-here in order (this note last, or first, as an index)
2. Read 01-manual-testing to build core testing judgment before automating anything
3. Pick ONE automation stack (02 Python or 03 TypeScript) rather than both at once
4. Move into 04-api-testing and 05-sql-database-testing once UI automation basics are solid
5. Only move into 07-09 (CI/CD, Git, Docker) once you're comfortable writing and
   maintaining test code — these are about scaling and delivering that code
6. Stages 3-4 folders can be read non-linearly based on interest/job requirements,
   but 19 (interview prep) and 23 (projects) are best used continuously alongside
   everything else, not saved for "the end"
```

## Production Considerations

- This note should be updated whenever a new top-level folder is added (per the repo's own rule of not adding folders without genuine long-term gaps) so it never falls out of sync with the actual repo structure.
- Keep this note free of topic-level detail — it's an index, not a summary. Each folder's own notes carry the depth; this just routes readers to them.

## Common Pitfalls

- Letting the roadmap go stale as new notes/folders are added — an outdated roadmap is worse than no roadmap, since it actively misdirects readers.
- Over-explaining individual topics here instead of linking to the actual notes — this file's job is navigation, not content.

## References

- N/A — this is an internal repo index, not based on external documentation.