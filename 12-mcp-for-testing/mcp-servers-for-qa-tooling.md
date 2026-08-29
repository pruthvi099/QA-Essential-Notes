# MCP Servers for QA Tooling

## What It Is

This note covers real, currently existing MCP servers relevant to QA/SDET work — primarily **Playwright MCP**, the official Microsoft-maintained server giving an AI agent control of a real, running browser through MCP's tool interface. This is grounded specifically in what exists and is documented today, not speculative future capability — consistent with this folder's standard of never overstating what an MCP integration actually does.

## Why It Matters

- Playwright MCP is a real, actively maintained, official tool (the `@playwright/mcp` package) — not a hypothetical or experimental side project — exposing 25+ tools for browser control through structured, LLM-friendly APIs. 
- Its core technical decision — using the accessibility tree instead of screenshots — has real, practical implications for reliability and cost that are worth understanding precisely, not just accepting as a marketing claim.
- The ecosystem here is genuinely fast-moving — Microsoft's own recommendation shifted in 2026 toward a Playwright CLI for coding-agent workflows specifically, citing significantly lower token usage per session than MCP  — which is itself an important, concrete lesson in how quickly guidance in this space can change, and why this note flags its currency explicitly.

## How It Works

**How Playwright MCP works technically:** it reads the page through the accessibility tree — the same semantic structure screen readers use — and returns structured data instead of screenshots,  rather than requiring a vision model to interpret pixel-based screenshots. This avoids the cost and latency of processing 500KB–2MB images per interaction that vision-model-based, screenshot-driven approaches require. 

**What it enables in practice:** the host application handles the model, the MCP server handles the browser, and the two communicate over JSON-RPC 2.0 — letting an AI navigate pages, click buttons, fill forms, and extract data  through the browser MCP exposes, without custom glue code per application.

**Two distinct use modes worth distinguishing:**
1. **Interactive/exploratory use** — an AI agent connected via MCP explores an application live, useful for exploratory-style investigation or generating a first-draft test.
2. **Test *authoring* assistance** — using the MCP-connected agent to help *write* actual Playwright Test code, which then becomes a normal, committed, deterministic test file — Playwright MCP accelerates exploration and test authoring, but committed Playwright Test code remains the durable regression artifact,  not the live MCP session itself.

## Example

Running the official Playwright MCP server locally, and its CI usage, showing the actual current tooling rather than a hypothetical:

```bash
# Adding the official server to an MCP-capable client (e.g., an IDE assistant)
npx @playwright/mcp@latest
```

```yaml
# Running Playwright MCP headlessly in CI (GitHub Actions), per current
# published guidance
- name: Run Playwright MCP
  run: npx @playwright/mcp@latest --headless --no-sandbox
```

A configuration file for more involved setups, reflecting the current schema:
```bash
npx @playwright/mcp@0.0.78 --config ./config/playwright-mcp.json
```

Configuration options include persistent profiles (for deliberate continuity across sessions), isolated profiles (for clean testing and CI), and extension mode for reusing an existing browser session when needed. 

**An important, current nuance worth stating precisely rather than glossing over:** as of 2026, Microsoft's own guidance recommends a Playwright CLI over MCP specifically for coding-agent workflows, citing roughly 4x lower token usage per session.  This means the "right tool" recommendation here is actively evolving — a genuine, current example of why this folder's caution against overstating fixed capabilities matters in practice.

## Production Considerations

- Treat MCP-assisted exploration/test-generation as a *drafting* tool, not a substitute for the durable, reviewed, committed test suite — the actual regression-safety value still comes from the checked-in Playwright Test code (see [Choosing a Test Framework From Scratch](../10-test-framework-design/choosing-a-test-framework-from-scratch.md)), not from live MCP sessions.
- Headless mode is typically the only supported option in standard Docker containers, and some features are off by default requiring manual activation  — verify actual behavior in your specific environment rather than assuming full feature parity between local and containerized runs.
- Given how quickly official guidance has shifted (MCP vs. CLI recommendations changing within the same year), verify current documentation before committing to either approach for a new project, rather than relying on any single source (including this note) as permanently current.

## Common Pitfalls

- Presenting MCP-based browser automation as a replacement for a maintained Playwright Test suite, when the actual, documented guidance is that it's a productivity layer on top of one, with the committed test code remaining the durable artifact.
- Assuming a specific MCP server's feature set or recommended usage pattern from months-old information — this is a genuinely fast-evolving area, and stated capabilities/recommendations have already changed materially within 2026 alone.
- Not distinguishing screenshot/vision-based browser automation tools from accessibility-tree-based ones (like Playwright MCP) — these have meaningfully different reliability and cost profiles, not just different implementation details.
- Overclaiming integration capability in professional/portfolio contexts (exactly what this folder's standard warns against) — describing a tool as tested and working when it hasn't actually been verified firsthand.

## Interview Notes

- Be ready to name Playwright MCP specifically as a real, current example of an MCP server relevant to QA work, and explain its accessibility-tree-based approach precisely.
- Understand the distinction between using MCP for interactive exploration/test authoring assistance versus the actual committed test suite that provides ongoing regression protection — this is the practical core of how MCP fits into real testing workflows today.
- Be able to acknowledge, honestly, that specific tool recommendations in this space are actively evolving — this shows current awareness rather than reciting stale, possibly-outdated guidance as fixed fact.

## References

- [Playwright MCP — GitHub](https://github.com/microsoft/playwright-mcp)
- [Playwright — Official Documentation](https://playwright.dev/)