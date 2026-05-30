# Platform Architecture Overview

> **Document Type:** Architecture Overview
> **Status:** Blueprint
> **Owner:** Platform Architecture Team
> **Last Updated:** 2026-05-30

---

## What Is the AI Operating Platform?

The AI Operating Platform is an enterprise AI control plane — a horizontal infrastructure layer that enables regulated organizations to deploy, govern, monitor, and evolve AI agents, workflows, and knowledge systems at scale.

It is **not** a chatbot. It is **not** an AI application. It is the **operating system** for AI — the layer between raw AI capabilities (models, embeddings, knowledge) and the business applications that need them.

---

## Platform in Context

```
┌──────────────────────────────────────────────────────────────────┐
│                    BUSINESS APPLICATIONS                          │
│  Loan Origination │ Claims Processing │ Risk Monitoring │ ...    │
│  (Built by tenant teams using Platform SDK/APIs)                 │
├──────────────────────────────────────────────────────────────────┤
│                  AI OPERATING PLATFORM                            │
│    [This is what we are building]                                │
│    Agents │ Knowledge │ Governance │ Observability │ Control      │
├──────────────────────────────────────────────────────────────────┤
│                  AI INFRASTRUCTURE                                │
│  Anthropic Claude │ OpenAI │ Qdrant │ Neo4j │ Ollama │ ...       │
└──────────────────────────────────────────────────────────────────┘
```

---

## The 16-Plane Model

The platform decomposes into 16 functional planes. Each plane has a single, well-defined responsibility.

```mermaid
graph TB
    subgraph Layer5["CONTROL LAYER"]
        P16["[16] Control Plane"]
        P15["[15] Developer Experience"]
    end

    subgraph Layer4["GOVERNANCE & TRUST"]
        P11["[11] Security"]
        P12["[12] Governance"]
        P13["[13] Trust"]
        P14["[14] Observability"]
    end

    subgraph Layer3["AGENT & ORCHESTRATION"]
        P10["[10] Evaluation"]
        P9["[9] Registry"]
        P8["[8] Workflow"]
        P7["[7] Decision"]
        P6["[6] Agent Runtime"]
    end

    subgraph Layer2["INTELLIGENCE"]
        P5["[5] Model Plane"]
        P4["[4] Semantic Plane"]
        P3["[3] Knowledge Plane"]
        P2["[2] Data Plane"]
    end

    subgraph Layer1["FOUNDATION"]
        P1["[1] Platform Foundation"]
    end

    Layer1 --> Layer2
    Layer2 --> Layer3
    Layer3 --> Layer4
    Layer4 --> Layer5
```

---

## Key Design Decisions (Summary)

| Decision | Choice | Reason |
|---|---|---|
| Platform language (services) | C# / ASP.NET Core | Enterprise maturity, type safety |
| AI runtime language | Python / FastAPI | AI ecosystem dominance |
| Agent framework | LangGraph | Stateful graphs, HITL, checkpointing |
| Tool protocol | MCP (Model Context Protocol) | Open standard, vendor neutral |
| Container orchestration | Kubernetes | Cloud agnostic, multi-tenant |
| Event streaming | Apache Kafka | Durable audit, replay, high throughput |
| Vector store | Qdrant | OSS, performance, filtering |
| Knowledge graph | Neo4j | Cypher, GDS, GraphRAG |
| Secrets | HashiCorp Vault | Dynamic secrets, PKI, multi-tenant |
| Observability | OpenTelemetry | Vendor neutral |
| Authorization | OPA (Rego) | Policy-as-code |
| Frontend | Next.js / React | Modern, type-safe |

All decisions are documented in full in [docs/adrs/](../adrs/).

---

## Data Flow (High Level)

```mermaid
sequenceDiagram
    participant App as Business Application
    participant API as Platform API
    participant Agent as Agent Runtime
    participant Model as Model Plane
    participant Knowledge as Knowledge Plane
    participant Gov as Governance Plane

    App->>API: Invoke AI capability
    API->>API: Authenticate + Authorize
    API->>Gov: Record operation started
    API->>Agent: Start agent run
    Agent->>Knowledge: Retrieve relevant knowledge (GraphRAG)
    Agent->>Model: Invoke model (with context)
    Model->>Model: Route to provider, apply rate limit
    Model-->>Agent: Model response
    Agent->>Gov: Record decision + lineage
    Agent-->>API: Result
    API-->>App: Governed AI response
    Gov->>Gov: Audit + compliance check (async)
```

---

## Non-Functional Architecture

### Performance
- Model invocation P95 latency: < 5 seconds
- RAG retrieval P95 latency: < 500ms
- Platform API P95 latency: < 200ms
- Agent step P95 latency: < 10 seconds

### Availability
- Platform API: 99.9% SLA
- Vault: 99.99% SLA (HA cluster required)
- Kafka: 99.9% SLA (3-broker replication)

### Scale
- Concurrent tenants: 50 (Standard tier)
- Model invocations per hour: 100,000
- Agent runs per hour: 5,000
- Knowledge base documents: 100 million

### Security
- All communication TLS 1.3
- All inter-service mTLS
- Zero static credentials (all dynamic via Vault)
- Zero-trust networking

### Compliance
- GDPR, SR 11-7, EU AI Act, HIPAA, SOX

---

## Where to Go Next

| Goal | Document |
|---|---|
| Understand the vision | [Platform Vision](../vision/platform-vision.md) |
| Review architecture principles | [Architecture Principles](../platform-principles/architecture-principles.md) |
| Deep-dive a specific domain | [Plane Documents](../planes/) |
| Understand a technology choice | [ADRs](../adrs/) |
| Implement a pattern | [Skills Documents](../skills/) |
| Plan implementation | [Roadmap](../roadmaps/roadmap-overview.md) |
| Reference architecture | [Reference Architectures](../reference-architectures/) |
