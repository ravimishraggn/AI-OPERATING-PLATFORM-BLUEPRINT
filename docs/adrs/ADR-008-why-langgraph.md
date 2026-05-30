# ADR-008 — Why LangGraph for Agent Orchestration

| Field | Value |
|---|---|
| **ADR Number** | 008 |
| **Title** | LangGraph as the Agent Orchestration Framework |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team, AI Engineering Team |
| **Supersedes** | None |

---

## Context

The Agent Runtime Plane must support:
- **Stateful agent execution** — agents that maintain state across many steps
- **Conditional branching** — agents that choose paths based on intermediate results
- **Human-in-the-loop** — agents that pause and request human approval
- **Multi-agent coordination** — supervisor agents delegating to specialist agents
- **Persistence** — agent state survives crashes and can be replayed
- **Observability** — every step, decision, and tool call traced
- **Governance** — policy evaluation at each step boundary

The framework must be Python-native (consistent with ADR-003), open-source, and designed for production enterprise use — not just prototype demos.

---

## Decision

**We will use LangGraph as the agent orchestration framework.**

LangGraph is used to define, compile, and execute agent graphs. The platform wraps LangGraph with:
- Custom checkpointers (Kafka-backed for durability)
- Governance middleware (policy evaluation at step boundaries)
- OpenTelemetry instrumentation at every node
- Tenant-aware execution context
- Platform-managed tool registry integration

---

## Alternatives Considered

### Alternative 1 — LangChain LCEL (LangChain Expression Language)
**Pros:**
- Integrated with LangChain ecosystem
- Chain-based composition is simple for linear workflows

**Cons:**
- Not designed for stateful, long-running agents
- No native human-in-the-loop support
- Limited branching and looping support
- LangGraph supersedes LCEL for complex agent patterns

**Verdict:** Rejected for agent orchestration. LangChain components (document loaders, retrievers) are used alongside LangGraph.

---

### Alternative 2 — CrewAI
**Pros:**
- High-level, easy to configure multi-agent workflows
- Good for role-based agent decomposition

**Cons:**
- Less flexible than LangGraph for complex conditional logic
- Less control over execution flow
- Less granular observability at step level
- Smaller enterprise adoption track record

**Verdict:** Rejected. LangGraph provides more control and enterprise-grade features.

---

### Alternative 3 — AutoGen (Microsoft)
**Pros:**
- Strong multi-agent conversation patterns
- Good for debate/critique agent patterns

**Cons:**
- Microsoft ecosystem; subtle lock-in risk
- Less flexible graph-based orchestration
- Human-in-the-loop less production-grade than LangGraph
- Checkpointing less mature

**Verdict:** Rejected. AutoGen patterns can be implemented within LangGraph if needed.

---

### Alternative 4 — Custom DAG Orchestration
**Pros:**
- Full control
- No framework dependency

**Cons:**
- Rebuilding what LangGraph already provides
- Significant engineering effort
- No community, no updates, no ecosystem

**Verdict:** Rejected. Framework investment is warranted; custom DAG is reinventing the wheel.

---

### Alternative 5 — Temporal (workflow orchestration)
**Pros:**
- Production-grade, highly durable workflow orchestration
- Built-in retry, timeout, and saga support
- Language-agnostic (Go server, Python/TypeScript/Java SDKs)

**Cons:**
- Not AI-specific; lacks LLM-native features (streaming, tool calling, memory)
- Requires separate Temporal server (additional service)
- Not a replacement for LangGraph; they serve different layers

**Verdict:** Not selected as agent framework. Temporal is considered for the Workflow Orchestration Plane (human workflows, business process orchestration) rather than AI agent execution.

---

## Why LangGraph

| Feature | LangGraph |
|---|---|
| Graph-based agent execution | Native |
| Stateful agents (short and long-term memory) | Native |
| Human-in-the-loop interrupts | Native |
| Conditional branching | Native |
| Streaming intermediate results | Native |
| Checkpointing (fault tolerance) | Native (pluggable checkpointers) |
| Multi-agent supervisor patterns | Native |
| Tool calling integration | Native |
| OpenTelemetry instrumentation | Via LangSmith / custom callbacks |
| Python-native | Yes |
| License | MIT (Open Source) |
| LangChain ecosystem integration | Full |

---

## LangGraph Architecture in Platform Context

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLATFORM AGENT RUNTIME                        │
├─────────────────────────────────────────────────────────────────┤
│  Agent Registry   │   Policy Middleware   │   Tenant Context    │
├─────────────────────────────────────────────────────────────────┤
│                        LANGGRAPH ENGINE                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Node 1   │──▶│  Node 2  │──▶│  Node 3  │──▶│  END     │    │
│  │ (Reason)  │   │ (Search) │   │ (Decide) │   │          │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Kafka Checkpointer │ OTEL Tracer │ MCP Tool Registry           │
└─────────────────────────────────────────────────────────────────┘
```

### Platform Extensions on Top of LangGraph

1. **Kafka Checkpointer:** Agent state checkpointed to Kafka for durability and replay
2. **Policy Middleware:** Governance policy evaluated at each node boundary
3. **OTEL Instrumentation:** Every node execution creates a trace span
4. **Tenant Context:** Tenant ID injected into all tool calls and state operations
5. **MCP Tool Registry:** Agent tools discovered and authorized from platform registry

---

## Consequences

### Positive
- Native support for complex, conditional, multi-step agent workflows
- Human-in-the-loop is a first-class feature (not an afterthought)
- Checkpointing enables fault-tolerant long-running agents
- Streaming enables real-time user interfaces for agent progress
- Graph-based visualization of agent execution

### Negative
- Python-only (consistent with ADR-003, but requires Python for all agent logic)
- LangGraph API has been evolving rapidly; version stability must be managed
- Platform must build Kafka checkpointer and governance middleware (not out-of-the-box)

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Native human-in-the-loop | Python-only constraint |
| Graph-based stateful agents | Framework API evolution risk |
| Checkpointing and fault tolerance | Custom Kafka checkpointer required |
| Rich multi-agent patterns | Governance middleware must be built on top |
