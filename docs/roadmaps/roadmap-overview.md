# Platform Roadmap Overview

> **Document Type:** Roadmap
> **Status:** Accepted
> **Owner:** Platform Architecture Team
> **Last Updated:** 2026-05-30

---

## Roadmap Philosophy

The platform is delivered in **phases**, not as a big-bang release. Each phase delivers independently valuable capabilities. No phase requires all previous phases to be 100% complete — there is planned parallelism within phases. Each phase ends with a capability milestone that can be demonstrated to stakeholders.

**Principle:** Deliver value early. Govern everything from the start. Never compromise on security, governance, or observability — even in Phase 1.

---

## Phase Overview

```mermaid
gantt
    title AI Operating Platform Delivery Roadmap
    dateFormat  YYYY-QQ
    axisFormat  %Y Q%q

    section Phase 1
    Architecture Blueprint         :done, p1, 2026-Q1, 2026-Q2

    section Phase 2
    Platform Foundation            :p2, 2026-Q2, 2026-Q3

    section Phase 3
    Knowledge Platform             :p3, 2026-Q3, 2026-Q4

    section Phase 4
    Agent Runtime                  :p4, 2026-Q4, 2027-Q1

    section Phase 5
    Governance and Trust           :p5, 2027-Q1, 2027-Q2

    section Phase 6
    Developer Experience           :p6, 2027-Q2, 2027-Q3

    section Phase 7
    Control Plane                  :p7, 2027-Q3, 2027-Q4

    section Phase 8
    Enterprise Readiness           :p8, 2027-Q4, 2028-Q2
```

---

## Phase Summary Table

| Phase | Name | Duration | Primary Output | Key Milestone |
|---|---|---|---|---|
| 1 | Architecture Blueprint | Q1-Q2 2026 | This repository | Blueprint complete, team aligned |
| 2 | Platform Foundation | Q2-Q3 2026 | Running Kubernetes platform | First service deployed to platform |
| 3 | Knowledge Platform | Q3-Q4 2026 | RAG + Knowledge Graph | First RAG query returns accurate result |
| 4 | Agent Runtime | Q4 2026 - Q1 2027 | Running governed agents | First governed agent completes a real task |
| 5 | Governance & Trust | Q1-Q2 2027 | Compliance-ready platform | First audit report generated automatically |
| 6 | Developer Experience | Q2-Q3 2027 | Self-service platform | First external team builds on platform independently |
| 7 | Control Plane | Q3-Q4 2027 | Full operator tooling | Platform fully operator-managed |
| 8 | Enterprise Readiness | Q4 2027 - Q2 2028 | Enterprise scale | Platform serves 10+ tenants in production |

---

## Cross-Phase Dependencies

```
Phase 1 (Blueprint) → All phases (foundation)
Phase 2 (Foundation) → Phase 3, 4, 5, 6, 7
Phase 3 (Knowledge) → Phase 4 (agents need knowledge)
Phase 4 (Agent Runtime) → Phase 5 (governance applied to agents)
Phase 4 + 5 → Phase 6 (DX needs something to surface)
Phase 6 + 7 → Phase 8 (enterprise readiness needs both)
```

---

## What Is NOT in the Roadmap

The roadmap deliberately excludes:
- Building specific business AI applications (loan decisioning, claims processing) — those are tenant responsibilities
- Training or fine-tuning foundational models
- Data collection outside AI operations
- Business intelligence or analytics platforms
- Replacing enterprise data warehouses

---

## Detailed Phase Documents

- [Phase 1 — Architecture Blueprint](./phase-01-architecture-blueprint.md)
- [Phase 2 — Platform Foundation](./phase-02-platform-foundation.md)
- [Phase 3 — Knowledge Platform](./phase-03-knowledge-platform.md)
- [Phase 4 — Agent Runtime](./phase-04-agent-runtime.md)
- [Phase 5 — Governance and Trust](./phase-05-governance-trust.md)
- [Phase 6 — Developer Experience](./phase-06-developer-experience.md)
- [Phase 7 — Control Plane](./phase-07-control-plane.md)
- [Phase 8 — Enterprise Readiness](./phase-08-enterprise-readiness.md)
