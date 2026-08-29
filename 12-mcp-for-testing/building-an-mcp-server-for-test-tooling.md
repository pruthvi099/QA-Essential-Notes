# Building an MCP Server for Test Tooling

## What It Is

This note covers the basic architecture of exposing a QA capability (running a test suite, fetching a report, querying test results) as a custom MCP server — using the same tools/resources/prompts primitives from [What Is MCP](./what-is-mcp.md), applied to a concrete, realistic testing use case: letting an AI agent trigger and inspect your own test infrastructure through a standardized interface.

## Why It Matters

- Building a custom MCP server is genuinely accessible with current SDKs (Python's official SDK, TypeScript equivalents) — this isn't a hypothetical, exotic capability, but a realistic project an SDET can actually build for internal tooling.
- The architectural guidance — one server per domain, rather than one monolithic server with every tool — matters directly for QA tooling, where test execution, reporting, and issue tracking are genuinely separate concerns worth separate servers.
- Given this folder's standard against overclaiming capability, this note focuses on the pattern and real, documented best practices rather than presenting a fully-built, untested "QA MCP server" as if it already exists as a polished product.

## How It Works

**Recommended domain separation:** the MCP server architecture is typically one server per domain — for QA tooling this suggests separate servers for, say, test execution, test reporting, and defect tracking, rather than one monolithic server handling everything — keeping tool sets small, descriptions focused, and the AI's context uncluttered.

**Core development workflow:**
1. Define tools (actions with side effects — running a test, filing a defect) and resources (read-only data — a stored report, a test run's status) using the appropriate SDK.
2. Write clear, careful tool descriptions — the AI uses these descriptions to decide which tool to call, so their quality directly determines whether the AI selects the right tool for a given request.
3. Test with the MCP Inspector — a browser-based tool that connects to the server and lets you call tools directly without an LLM layer, letting you verify behavior before ever connecting a real AI client.
4. Choose the appropriate transport: stdio for local development/tools (low latency, ideal for a desktop AI assistant), Streamable HTTP for anything deployed remotely or shared across a team.

## Example

A minimal Python MCP server exposing a test-execution tool and a report resource, following current best practices (clear descriptions, structured output, input validation):

```python
from mcp.server.fastmcp import FastMCP
import subprocess
import json

mcp = FastMCP("QA-Test-Runner")

@mcp.tool()
def run_test_suite(suite_name: str, environment: str = "staging") -> dict:
    """
    Run a named Playwright test suite against a specified environment
    and return structured pass/fail results.

    Args:
        suite_name: The test suite tag to run (e.g., "smoke", "regression")
        environment: Target environment - "staging" or "production"
    """
    # Clear description above is what the AI uses to decide WHEN to call
    # this tool — vague descriptions lead to the AI misusing or ignoring it
    if environment not in ("staging", "production"):
        return {"error": f"Invalid environment: {environment}"}

    result = subprocess.run(
        ["npx", "playwright", "test", f"--grep=@{suite_name}", "--reporter=json"],
        capture_output=True, text=True, env={"TEST_ENV": environment}
    )
    # Return STRUCTURED output the AI can reason about, not raw text
    return {
        "suite": suite_name,
        "environment": environment,
        "exit_code": result.returncode,
        "passed": result.returncode == 0,
    }

@mcp.resource("test-report://{run_id}")
def get_test_report(run_id: str) -> str:
    """Fetch a previously generated test report by run ID (read-only, no side effects)."""
    with open(f"reports/{run_id}.json") as f:
        return f.read()
```

Testing this server locally with the MCP Inspector before connecting any real AI client — verifying actual behavior, not assuming it:
```bash
npx @modelcontextprotocol/inspector python qa_server.py
# Opens a local browser UI to call run_test_suite() and get_test_report()
# directly, confirming they behave correctly before any AI agent uses them
```

## Production Considerations

- Always validate tool inputs server-side using strict schemas (Pydantic in Python, Zod in TypeScript) — never trust an LLM to emit well-formed arguments, since hallucinated or malformed parameter names are a documented, real failure mode.
- Never put credentials (API keys, database passwords) directly in tool schemas or resource payloads — store them in environment variables on the server side, the same principle as [Secrets & Environment Management in CI](../07-ci-cd/secrets-and-environment-management-in-ci.md) applied to MCP server configuration.
- Log every tool invocation with timestamps, arguments (sanitized for PII), and results — this creates the audit trail needed both for debugging AI-driven tool usage and for the compliance/security expectations increasingly required for enterprise AI agent deployments.

## Common Pitfalls

- Writing vague or minimal tool descriptions, causing the AI to call the wrong tool or fail to use an available tool appropriately — description quality has a direct, outsized effect on real usability.
- Building one large, monolithic server exposing every QA capability at once instead of separate, domain-focused servers — this makes tool sets harder for the AI to reason about and increases the blast radius of any single server's compromise.
- Skipping server-side