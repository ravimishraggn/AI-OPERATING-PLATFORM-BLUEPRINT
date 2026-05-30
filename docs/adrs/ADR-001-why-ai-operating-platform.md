# ADR-001 — Why Build an AI Operating Platform

| Field | Value |
|---|---|
| **ADR Number** | 001 |
| **Title** | Why Build an AI Operating Platform Rather Than AI Applications |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team, CTO Office |
| **Supersedes** | None |

---

## Context

The organization needs to deploy AI capabilities across multiple business domains including risk management, customer service, document intelligence, and regulatory compliance. The question is: should we build separate AI applications for each domain, or build a shared AI operating platform?

Currently, multiple teams are independently building AI solutions using different technologies, different providers, different security approaches, and different governance mechanisms. This results in:

- Duplicated infrastructure and integration effort
- Inconsistent security and governance posture
- No cross-team knowledge sharing or reuse
- Multiple vendor relationships with no leverage
- No organizational visibility into AI operations
- Regulatory exposure from ungoverned AI deployments

The choice is between:
1. **Point Solutions:** Each team builds their own AI stack
2. **AI Platform:** A shared operating platform consumed by all teams

---

## Decision

**We will build an Enterprise AI Operating Platform as a shared, horizontal platform layer.**

This platform acts as an AI Control Plane — a governance, operations, and capability layer that sits between raw AI infrastructure (models, vector stores, knowledge graphs) and the business applications that need AI capabilities.

Business teams build applications on top of the platform. They do not build AI infrastructure.

---

## Alternatives Considered

### Alternative 1 — Team-by-Team Point Solutions
**Approach:** Each business team independently selects, integrates, and operates their own AI stack.

**Pros:**
- Maximum team autonomy
- Faster initial delivery for first team
- No platform dependency

**Cons:**
- Exponential duplication of effort across teams
- No consistent governance or security
- Regulatory exposure accumulates with each deployment
- Knowledge and tooling cannot be shared
- Organizational AI maturity cannot improve systematically
- Cost grows linearly with teams (no economies of scale)

**Verdict:** Rejected. Does not scale. Creates ungovernable AI sprawl.

---

### Alternative 2 — Buy a Commercial AI Platform
**Approach:** Purchase a commercial AI platform (Microsoft Azure AI Studio, AWS SageMaker, Google Vertex AI, Scale AI, Dataiku).

**Pros:**
- Faster to initial capability
- Vendor-managed infrastructure

**Cons:**
- Significant vendor lock-in
- High and unpredictable cost at enterprise scale
- Limited customization for regulated industry requirements
- Data sovereignty concerns (data leaves the organization)
- Governance controls may not meet regulatory requirements
- Cannot run on-premises without hybrid complexity
- Provider-specific APIs propagate through the organization

**Verdict:** Rejected as primary strategy. Commercial tools may be used as integration targets behind abstraction layers.

---

### Alternative 3 — Thin Integration Layer Only
**Approach:** Build a minimal API gateway over AI providers without a full platform.

**Pros:**
- Lower initial investment
- Simpler to build

**Cons:**
- Does not address governance, audit, evaluation, or knowledge management
- Does not provide agent runtime capabilities
- Cannot support regulated-industry requirements
- No path to agentic AI at scale

**Verdict:** Rejected. Insufficient for enterprise AI operations.

---

## Consequences

### Positive
- **Governance at scale:** Every AI operation governed centrally through policy
- **Reuse:** Common capabilities (RAG, knowledge, evaluation) built once and shared
- **Security:** Consistent security posture across all AI workloads
- **Auditability:** Full audit trail for regulatory compliance
- **Cost efficiency:** Shared infrastructure reduces per-team cost
- **Provider agility:** Model providers can be swapped without business logic changes
- **Organizational learning:** Evaluation data flows to a single platform; AI quality improves systemically

### Negative
- **Initial investment:** Significant architecture and engineering effort before business teams can deliver
- **Platform dependency:** Business teams depend on platform team delivery velocity
- **Complexity:** Platform architecture is significantly more complex than point solutions
- **Governance overhead:** Every AI feature requires platform integration, not just code

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Platform becomes a bottleneck | Self-service model; teams deploy within platform guardrails without platform team approval |
| Platform adoption is slow | Developer experience investment; platform must be easier to use than building direct |
| Platform over-engineering | Phase-gated delivery; build the minimum that meets the next phase's needs |

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Consistent governance | Initial delivery speed |
| Provider agility | Platform learning curve |
| Shared capabilities | Platform dependency |
| Regulatory compliance | Engineering investment |
| Organizational AI maturity | Architectural complexity |

---

## Review

This ADR will be reviewed at the end of Phase 2 (Platform Foundation) to confirm the approach is delivering the expected value. If adoption is below 50% of target teams by Month 12, a course correction will be initiated.
