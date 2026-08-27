# What Is MCP (Model Context Protocol)

## What It Is

The Model Context Protocol (MCP) is an open standard for connecting AI applications (like an LLM-powered assistant) to external tools and data sources through a common, standardized interface — rather than every AI application needing a custom, one-off integration with every tool it wants to use. MCP defines a **client-server** architecture with three core primitives: **tools** (actions the model can invoke), **resources** (data the application can fetch and provide as context), and **prompts** (reusable templates a user selects to guide an interaction). 

This note is specifically about the protocol itself, not about "AI agents" generally — MCP is one standardized way agentic AI systems connect to the outside world, not a synonym for agentic AI as a whole.

## Why It Matters

- MCP addresses a real, practical integration problem: without a shared standard, connecting N AI applications to M different tools/data sources requires N×M custom integrations — MCP aims to reduce this to a standardized interface each side implements once. 
- As QA tooling (test runners, issue trackers, browser automation) increasingly gets exposed via MCP servers, understanding the protocol's actual architecture — not just "AI can use tools now" — is what lets an SDET evaluate a specific MCP-based tool's real capabilities and limits.
- Given the master prompt's explicit standard for this folder — never exaggerate MCP capabilities, never claim an integration exists unless it does — precision about what MCP actually is (and isn't) matters more here than in most other topics in this repo.

## How It Works

**Client-server architecture:** an MCP client lives within a host application and maintains a connection to one specific MCP server, and servers are external programs that expose tools, resources, and prompts to the AI model via the client.  A host application (like an IDE assistant or a chat client) can run multiple clients, each connected to a different server.

**The three core primitives, and who controls each:**
- **Tools** — functions the LLM can call to perform specific actions (e.g., a weather API call) — this is essentially function calling, and tools are model-controlled. 
- **Resources** — data sources the LLM can access, similar to GET endpoints in a REST API, providing data without side effects;  resources are application-driven — the application decides when to fetch and pass them as context, unlike tools which the model itself decides to invoke. 
- **Prompts** — pre-defined templates for using tools/resources effectively, selected by the user before running inference,  and these can be versioned and updated centrally on the server without requiring any client-side code changes. 

**Servers enforce their own security boundaries** — a server encapsulates a domain-specific responsibility (a filesystem, a database, a network API) and enforces security constraints so clients only access authorized resources or operations. 

## Example

A minimal MCP server (Python), showing a tool and a resource concretely, illustrating the core distinction between them:

```python
from mcp.server import MCPServer

mcp = MCPServer("QA-Demo-Server")

@mcp.tool()
def run_smoke_test(test_name: str) -> str:
    """Execute a named smoke test and return its pass/fail result."""
    # In a real server, this would actually invoke the test runner
    return f"Test '{test_name}' result: PASS"

@mcp.resource("test-report://{run_id}")
def get_test_report(run_id: str) -> str:
    """Fetch a previously generated test report by run ID."""
    # In a real server, this would fetch an actual stored report
    return f"Report for run {run_id}: 42 passed, 0 failed"
```

This is a complete, minimal MCP server — one tool, one templated resource — that can be inspected and tested with the official MCP Inspector tool before being connected to a real host application. 

The conceptual distinction this illustrates: `run_smoke_test` is something the model decides to *invoke* (an action, with a side effect — a test actually runs); `get_test_report` is something the application fetches to *provide as context* (read-only data, no side effect) — this tool-vs-resource distinction is one of the most fundamental, and most commonly tested, MCP concepts.

## Production Considerations

- Recommended transport for production MCP deployments is HTTP-based (such as Streamable HTTP), not the stdio transport — stdio transport is intended only for local server connections and cannot be deployed to production environments. 
- Security depends on explicit user consent at the protocol level — clients must request explicit permission before accessing tools or resources,  though this protection's real-world effectiveness depends on users actually understanding what they're approving, not just the existence of a permission prompt.
- Server-side boundary configuration (restricting what a server can actually access — a specific directory, a specific dataset) is the current recommended approach for scoping a server's access, since the earlier "Roots" mechanism for client-side boundary declaration has been deprecated in favor of boundary configuration on the server side through tool parameters, resource URLs, or server settings. 

## Common Pitfalls

- Treating "MCP" as a general synonym for "AI agent" or "AI tool use" broadly — MCP is a specific, standardized protocol for that connection, not the concept of agentic AI itself.
- Confusing tools (model-controlled actions with potential side effects) with resources (application-controlled, read-only data access) — this distinction matters for understanding what a given MCP server capability actually does and who decides when it fires.
- Overstating what a specific MCP integration actually does — claiming a tool exists or has been tested when it hasn't, which the repo's own standard for this folder explicitly warns against.
- Assuming a permission prompt alone guarantees safe usage — the real security value depends on the user genuinely understanding what they're granting access to, not just that a prompt technically appeared.

## Interview Notes

- Be ready to explain the three core MCP primitives (tools, resources, prompts) and precisely who controls each (model, application, user respectively) — a foundational, specific distinction worth stating exactly.
- Understand the client-server architecture and the N×M integration problem MCP is designed to reduce — this is the core "why" behind the protocol's existence.
- Be able to explain why claiming an untested MCP integration "works" is a real credibility risk in this space specifically, given how new and fast-moving the ecosystem is — this repo's own standard reflects a broader, warranted caution in the field.

## References

- [Model Context Protocol — Official Specification](https://modelcontextprotocol.io/)
- [Model Context Protocol — Official Python SDK](https://github.com/modelcontextprotocol/python-sdk)