# ADR-009 — Why Model Context Protocol (MCP)

| Field | Value |
|---|---|
| **ADR Number** | 009 |
| **Title** | Model Context Protocol as the Standard for Agent Tool Integration |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

AI agents need to interact with external systems: databases, APIs, file systems, services, and specialized tools. Without a standard protocol, each agent-tool integration is custom — creating a maintenance explosion as agents and tools multiply.

Requirements for the agent tool integration protocol:
- **Standardized:** All tools expose the same interface regardless of vendor
- **Discoverable:** Agents can discover available tools at runtime
- **Typed:** Tool inputs and outputs are schema-defined
- **Governable:** Tool access is authorized and auditable
- **Provider neutral:** Works with any AI model (Claude, GPT, Gemini, local)
- **Open:** Not owned by a single vendor

---

## Decision

**We will use the Model Context Protocol (MCP) as the standard protocol for agent-tool integration.**

All platform tools are exposed as MCP servers. All agents consume tools via the MCP client interface. The platform maintains an MCP Tool Registry that agents query to discover available, authorized tools.

---

## What Is MCP

MCP (Model Context Protocol) is an open protocol introduced by Anthropic that defines a standard way for AI models and agents to interact with external tools, resources, and services. It provides:

- A JSON-RPC 2.0 based protocol
- Tool discovery (list available tools)
- Tool invocation (call a tool with typed arguments)
- Resource access (read structured data from external systems)
- Prompt management (reusable prompt templates)
- Sampling (request model inference from the MCP server)

MCP is model-agnostic: it works with Claude, GPT, Gemini, and any MCP-capable agent framework.

---

## Alternatives Considered

### Alternative 1 — OpenAI Function Calling / Tool Use
**Pros:**
- Widely supported by GPT-4 and compatible models
- Simple JSON schema for tool definitions

**Cons:**
- OpenAI-specific format (though other providers have adopted similar formats)
- No standardized discovery mechanism
- No transport-level protocol (tools are defined inline, not as servers)
- Violates Vendor Neutral principle (P6) if used as the platform standard

**Verdict:** Not adopted as platform standard. OpenAI-compatible tool formats can be supported within MCP adapters.

---

### Alternative 2 — Custom Platform Tool Protocol
**Pros:**
- Full control
- Tailored to platform needs

**Cons:**
- Every AI framework must be updated to support the custom protocol
- No community, no ecosystem, no model provider support
- Significant engineering investment to maintain

**Verdict:** Rejected. Standardization value of MCP is too high to build a competing protocol.

---

### Alternative 3 — REST APIs (direct service calls from agents)
**Pros:**
- Simple; existing APIs immediately usable
- No protocol overhead

**Cons:**
- No discovery (agents must have tool definitions hardcoded)
- No governance at protocol level
- No standardized error handling
- Every tool is a custom integration

**Verdict:** Rejected as platform standard. REST endpoints are exposed as MCP tools via MCP server wrappers.

---

### Alternative 4 — LangChain Tools
**Pros:**
- Rich existing tool ecosystem
- Integrated with LangGraph

**Cons:**
- LangChain-specific; not portable to other frameworks
- No standard transport protocol
- No discovery server model
- Vendor-specific (LangChain/LangSmith)

**Verdict:** LangChain tools can be wrapped as MCP tools. Not adopted as the standalone standard.

---

## Why MCP Wins

| Feature | MCP |
|---|---|
| Open standard | Yes (Anthropic-initiated, community-governed) |
| Provider agnostic | Works with any model |
| Standardized discovery | Tool listing via protocol |
| Typed schemas | JSON Schema for inputs/outputs |
| Transport options | stdio, HTTP SSE, WebSocket |
| Growing ecosystem | 1000s of community MCP servers |
| LangGraph integration | Native (via MCP client) |
| Platform-level governance | Authorization per tool, per agent |
| IDE integration (Claude Code) | Native |
| Multi-model support | Claude, GPT, Gemini, Ollama |

---

## MCP in the Platform

### MCP Server Registry
The platform runs an **MCP Tool Registry** — a catalog of all available MCP servers (tools) the platform exposes. Agents query the registry to discover tools they are authorized to use.

```
Agent Request → MCP Client → Tool Registry Lookup → 
Authorization Check (Governance Plane) → MCP Server Call → 
Result → Audit Log → Agent
```

### Platform MCP Servers (Examples)
```
mcp://platform/knowledge-graph      ← Neo4j query tool
mcp://platform/vector-search        ← Qdrant semantic search
mcp://platform/document-retrieval   ← RAG pipeline
mcp://platform/data-query           ← PostgreSQL query (governed)
mcp://platform/workflow-trigger     ← Trigger business workflows
mcp://platform/human-review         ← Request human review
mcp://platform/audit-write          ← Write governance record
```

### Authorization Model
Each agent has a declared tool scope:
```json
{
  "agent_id": "loan-underwriting-agent",
  "allowed_tools": [
    "mcp://platform/knowledge-graph",
    "mcp://platform/vector-search",
    "mcp://platform/human-review"
  ]
}
```

Any tool call outside the declared scope is blocked and audited.

---

## Consequences

### Positive
- All tools behind a standard interface — agents don't know or care what's behind the tool
- Tool discovery enables dynamic agent composition
- Authorization at the MCP layer enables governance without agent code changes
- Growing ecosystem of community MCP servers immediately usable
- Works with Claude Code, Claude desktop, and any MCP client

### Negative
- MCP is a relatively new protocol (2024); ecosystem still maturing
- Adds a server hop for all tool calls (latency consideration)
- MCP server management is an operational concern

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Open, vendor-neutral standard | Protocol immaturity (evolving spec) |
| Tool discovery and authorization | MCP server per tool (operational overhead) |
| Provider-agnostic | Additional latency per tool call |
| Growing community ecosystem | Implementation effort for all platform tools |
