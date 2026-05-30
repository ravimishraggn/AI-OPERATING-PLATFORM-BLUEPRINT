# CLAUDE.md — AI Operating Platform Blueprint

This file provides architectural guidance to Claude Code when working within this repository.

## What This Repository Is

This is the **architecture blueprint** for an Enterprise AI Operating Platform. It is a documentation-first repository. No production code exists here. The purpose is to define the platform's structure, decisions, contracts, and standards so that implementation teams can build confidently.

## What Claude Should Do Here

- Generate and refine **architecture documentation**
- Write and maintain **ADR documents** following the established format
- Produce **Mermaid diagrams** for architecture visualization
- Write detailed **plane documents** following the 21-section template
- Help navigate and cross-reference documents within the repository
- Suggest improvements to architecture based on patterns in `docs/skills/`

## What Claude Should NOT Do Here

- Do not generate production application code
- Do not create deployment scripts or CI/CD pipelines
- Do not implement business logic
- Do not create database schemas or migration files
- Do not write test files
- Do not generate infrastructure-as-code (Terraform, Helm, etc.) unless explicitly requested as a blueprint artifact

## Key Architecture Principles (Always Apply)

1. **Open Source First** — prefer OSS solutions, document commercial alternatives
2. **Provider Agnostic** — all AI provider integrations use abstraction layers
3. **Vendor Neutral** — no component is tightly coupled to a single vendor
4. **Security First** — every document must address security implications
5. **Governance First** — every AI capability must have governance hooks
6. **Cloud Agnostic** — all designs work on any cloud or on-premises

## Technology Stack

When generating architecture content, always reference these as the target stack:

- **Platform Services:** C# / ASP.NET Core
- **AI Runtime:** Python / FastAPI / LangGraph
- **Frontend:** React / Next.js / TypeScript
- **Containers:** Docker / Kubernetes
- **Messaging:** Apache Kafka
- **Database:** PostgreSQL
- **Cache:** Redis
- **Vector DB:** Qdrant
- **Knowledge Graph:** Neo4j (primary) / Kuzu (embedded)
- **Secrets:** HashiCorp Vault
- **Observability:** OpenTelemetry, Prometheus, Grafana, Loki, Jaeger
- **Agent Protocol:** MCP (Model Context Protocol)

## Document Standards

### Plane Documents (docs/planes/)
All plane documents must follow the 21-section template:
1. Purpose
2. Business Problem
3. Responsibilities
4. Non-Responsibilities
5. Architecture Overview
6. Components
7. Internal Services
8. APIs
9. Data Flow
10. Security Requirements
11. Observability Requirements
12. Scalability Considerations
13. Multi-Tenant Considerations
14. Future Roadmap
15. Dependencies
16. Risks
17. Tradeoffs
18. Technology Choices
19. Alternatives Considered
20. Sequence Diagrams (Mermaid)
21. Architecture Diagrams (Mermaid)

### ADR Documents (docs/adrs/)
All ADRs must follow this format:
- **Status:** Proposed | Accepted | Deprecated | Superseded
- **Context:** The situation requiring a decision
- **Decision:** The chosen approach
- **Alternatives Considered:** What else was evaluated
- **Consequences:** What this decision implies
- **Tradeoffs:** What we gain and lose

### Reference Architecture Documents (docs/reference-architectures/)
Must include:
- Executive summary (2-3 paragraphs)
- Architecture goals
- Constraints
- Component diagram (Mermaid)
- Data flow diagram (Mermaid)
- Sequence diagram (Mermaid)
- Non-functional requirements
- Integration points
- Security model

## Naming Conventions

- Plane docs: `NN-plane-name.md` (e.g., `01-platform-foundation.md`)
- ADRs: `ADR-NNN-short-title.md` (e.g., `ADR-001-why-ai-operating-platform.md`)
- Reference architectures: `descriptive-name.md`
- Skills: `domain-patterns.md`

## Multi-Tenant Model

The platform uses a **namespace-per-tenant** model in Kubernetes, with shared infrastructure services and isolated data stores. When designing any component, always consider:
- Tenant identification (tenant ID on all requests/events)
- Data isolation (row-level security, separate schemas, or separate stores)
- Resource quotas (CPU, memory, API rate limits per tenant)
- Audit separation (tenant-scoped audit logs)

## Security Model

All designs must follow:
- **Zero-trust:** No implicit trust between services
- **Least privilege:** Minimum permissions required
- **Encryption at rest and in transit**
- **Key management via HashiCorp Vault**
- **Audit logging for all AI operations**
- **Data sovereignty controls** (data residency, cross-border restrictions)

## AI Provider Abstraction

All AI model calls must go through the **Model Plane** abstraction layer. Never reference a specific AI provider SDK directly in architectural designs — always use the platform's model router interface.

```
Client → Model Router → Provider Adapter → [Claude | OpenAI | Bedrock | Gemini | Ollama]
```

## Glossary Reference

When using domain-specific terms, ensure they are defined in `docs/glossary/glossary.md`.

## Roadmap Alignment

All architecture decisions should align with the phased roadmap in `docs/roadmaps/`. Note:
- Phase 1: Architecture Blueprint (current)
- Phase 2: Platform Foundation
- Phase 3: Knowledge Platform
- Phase 4: Agent Runtime
- Phase 5: Governance & Trust
- Phase 6: Developer Experience
- Phase 7: Control Plane
- Phase 8: Enterprise Readiness
