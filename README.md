# QA-Essential-Notes

A practical, in-depth QA/SDET knowledge base — from manual testing fundamentals to advanced automation architecture, framework design, and interview preparation. Written the way an experienced SDET would explain concepts to another engineer: real code, real trade-offs, no filler.

**24 topic areas · growing daily · Python & TypeScript covered side by side**

New here? Start with the **[Roadmap](./00-start-here/roadmap.md)** for a guided path from beginner to advanced SDET.

---

## Why this repo

Most QA/testing content online is either too shallow (definitions with no code) or too narrow (a single tool's docs). This repo aims for both depth and breadth:

- **What it is → Why it matters → How it works → Example → Production considerations → Common pitfalls → Interview notes → References** — every note follows the same rigorous structure
- **Working code**, not pseudocode — Python and TypeScript Playwright examples side by side wherever both apply
- **Production-oriented**, not just "how to pass a test" — covers flakiness, CI/CD, framework architecture, and real trade-offs
- **Interview-relevant** — every note ends with what an interviewer would actually probe on

## A few notes to start with

- [Page Object Model](./02-automation-python-playwright/page-object-model.md) — the most commonly asked SDET framework question, explained with working Python + TS examples
- [Flaky Test Handling](./02-automation-python-playwright/flaky-test-handling.md) — a systematic diagnostic framework, not just "add a retry"
- [QA Engineer vs. SDET](./00-start-here/qa-engineer-vs-sdet.md) — the distinction most confuse, with a concrete before/after example
- [Risk-Based Testing](./00-start-here/risk-based-testing.md) — how to actually prioritize test effort under real time constraints

## Repository Structure

| Folder | Covers |
|---|---|
| [00-start-here](./00-start-here/) | Core QA vocabulary, SDLC/STLC, test design techniques, roles |
| [01-manual-testing](./01-manual-testing/) | Exploratory testing, defect reporting, usability, accessibility |
| [02-automation-python-playwright](./02-automation-python-playwright/) | Playwright automation — Python & TypeScript side by side |
| [03-typescript-playwright](./03-typescript-playwright/) | TypeScript-specific: types, generics, mocking, visual regression |
| [04-api-testing](./04-api-testing/) | REST, auth, schema validation, contract testing |
| [05-sql-database-testing](./05-sql-database-testing/) | Backend/data validation |
| [06-mobile-testing](./06-mobile-testing/) | Mobile app testing |
| [07-ci-cd](./07-ci-cd/) | Pipelines, quality gates |
| [08-git-github](./08-git-github/) | Version control workflows for test code |
| [09-docker](./09-docker/) | Containerized test environments |
| [10-test-framework-design](./10-test-framework-design/) | Architecture, patterns, scalability |
| [11-ai-testing](./11-ai-testing/) | AI-assisted testing practices |
| [12-mcp-for-testing](./12-mcp-for-testing/) | Model Context Protocol for QA tooling |
| [13-security-testing](./13-security-testing/) | Security fundamentals for testers |
| [14-performance-testing](./14-performance-testing/) | Load, stress, and performance testing |
| [15-accessibility-visual-testing](./15-accessibility-visual-testing/) | Automated a11y and visual regression |
| [16-observability-debugging](./16-observability-debugging/) | Logs, traces, debugging distributed systems |
| [17-cloud-testing](./17-cloud-testing/) | Testing in cloud environments |
| [18-quality-engineering](./18-quality-engineering/) | Broader quality engineering practices |
| [19-sdet-interview-preparation](./19-sdet-interview-preparation/) | Interview prep |
| [20-real-world-scenarios](./20-real-world-scenarios/) | Applied scenarios |
| [21-testing-strategies](./21-testing-strategies/) | Strategy and planning |
| [22-system-design-for-sdet](./22-system-design-for-sdet/) | SDET-focused system design |
| [23-projects](./23-projects/) | End-to-end applied projects |

See the [full roadmap](./00-start-here/roadmap.md) for the intended reading order and growth stages.

## Who this is for

- **Beginners** learning QA from the ground up
- **Manual testers** moving into automation
- **QA Automation Engineers** growing toward SDET
- **Recruiters/interviewers** evaluating SDET candidates' depth
- **Anyone** who wants a single, well-organized reference instead of scattered blog posts

## Contributing

This is currently a personal, actively maintained knowledge base — new notes are added regularly. Found an error or have a suggestion? Open an issue.

## License

MIT — free to use, reference, and share.♥️💻🔁