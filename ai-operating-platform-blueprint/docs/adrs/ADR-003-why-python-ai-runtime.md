# ADR-003 — Why Python for AI Runtime

| Field | Value |
|---|---|
| **ADR Number** | 003 |
| **Title** | Python / FastAPI for AI Runtime Layer |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

The AI runtime layer handles all AI-specific execution: model invocation, embedding generation, vector search orchestration, RAG pipeline execution, LangGraph agent execution, MCP server implementation, and knowledge graph queries for AI context.

This layer is distinct from the platform services layer (C# — ADR-002). The AI runtime is specialized for AI workloads; the platform services are specialized for enterprise control plane functions.

The AI runtime must:
- Execute LangGraph agent graphs (workflows)
- Invoke AI models via provider adapters
- Generate and search embeddings in Qdrant
- Query Neo4j knowledge graphs for context
- Serve as MCP server endpoints for agent tool use
- Process RAG pipelines with document chunking and retrieval
- Expose async HTTP APIs for platform service consumption

---

## Decision

**We will use Python 3.12+ with FastAPI for the AI Runtime layer.**

---

## Alternatives Considered

### Alternative 1 — C# (same as platform services)
**Pros:**
- Single language across the platform
- Strong type system
- Excellent performance

**Cons:**
- C# AI ecosystem is significantly less mature than Python
- LangGraph is Python-only (no C# port)
- Most AI/ML libraries (transformers, sentence-transformers, RAG tooling) are Python-first
- Qdrant Python client is the reference client; .NET client is a community port
- AI engineers universally work in Python; C# would reduce talent pool
- Embedding models (local) run through Python (HuggingFace, Sentence Transformers)

**Verdict:** Rejected. C# cannot access the AI ecosystem with the same depth as Python.

---

### Alternative 2 — JavaScript / TypeScript (Node.js)
**Pros:**
- Shared language with frontend
- LangChain.js exists
- Good async model

**Cons:**
- AI/ML ecosystem significantly smaller than Python
- LangGraph has no official JS port
- Local model inference not supported
- AI engineers do not typically work in TypeScript
- Performance limitations for CPU-bound tasks

**Verdict:** Rejected. Not viable for AI runtime.

---

### Alternative 3 — Julia
**Pros:**
- Excellent numerical computing performance
- Growing ML ecosystem

**Cons:**
- Negligible enterprise adoption
- No LangGraph, LangChain equivalents
- No MCP tooling
- No available talent pool

**Verdict:** Rejected. Wrong ecosystem.

---

## Why Python Wins

| Criterion | Value |
|---|---|
| LangGraph | Python-native; the definitive choice for graph-based agent orchestration |
| LangChain | Richest AI toolchain; Python-first |
| MCP SDK | Python reference implementation |
| Qdrant client | Python reference client |
| Neo4j driver | First-class Python support |
| Embedding models | HuggingFace Transformers, Sentence Transformers — Python-only |
| AI talent pool | Python dominates AI engineering globally |
| FastAPI | Async, type-annotated, OpenAPI-first — ideal for AI service APIs |
| Async I/O | asyncio / uvloop handles concurrent model calls efficiently |

---

## FastAPI Specifically

FastAPI is selected over Flask, Django REST Framework, and Tornado because:
- Native `async def` route handlers — critical for concurrent AI calls
- Automatic OpenAPI / Swagger generation from type annotations
- Pydantic v2 for request/response validation (fast Rust-backed validation)
- Background tasks API
- WebSocket support for streaming model responses
- Low overhead — performance competitive with Node.js
- Built-in dependency injection for service wiring

---

## Consequences

### Positive
- Full access to the Python AI ecosystem
- LangGraph agent orchestration natively supported
- FastAPI's async model handles concurrent model calls without threading complexity
- Streaming responses supported natively (Server-Sent Events, WebSocket)
- AI engineers can contribute directly without language barrier

### Negative
- Two language ecosystems (C# + Python) require polyglot team management
- Python packaging can be complex (poetry, pyproject.toml management)
- Python type annotations are optional; discipline required for consistency
- Container images larger than Go/Rust equivalents

### Mitigations
- Strict use of `uv` or `poetry` for dependency management
- Mandatory type annotations enforced via `mypy` in CI
- Slim container images using multi-stage builds
- Clear API contracts between C# and Python services via OpenAPI and Protobuf

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Full AI ecosystem access | Polyglot team requirement |
| LangGraph native support | Python packaging complexity |
| AI talent availability | Type safety requires discipline |
| Fastest path to production AI | Two language ecosystems |
