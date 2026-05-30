# Phase 2 — Platform Foundation

> **Phase:** 2 of 8
> **Status:** Planned
> **Timeline:** Q2-Q3 2026
> **Owner:** Platform Infrastructure Team

---

## Goals

1. Deploy a production-grade Kubernetes platform (cloud and on-premises)
2. Establish the secrets management infrastructure (HashiCorp Vault)
3. Implement GitOps deployment pipeline (Argo CD)
4. Deploy core data infrastructure (PostgreSQL, Kafka, Redis)
5. Deploy vector and graph stores (Qdrant, Neo4j)
6. Implement basic observability (OpenTelemetry, Prometheus, Grafana)
7. Deploy the Model Plane (AI model routing, provider adapters)
8. Achieve first end-to-end model invocation through the platform

---

## Deliverables

| Deliverable | Component | Priority |
|---|---|---|
| Kubernetes cluster (cloud) | Platform Foundation | P0 |
| Kubernetes cluster (on-premises config) | Platform Foundation | P0 |
| HashiCorp Vault HA cluster | Platform Foundation | P0 |
| Argo CD GitOps pipeline | Platform Foundation | P0 |
| cert-manager + PKI setup | Platform Foundation | P0 |
| Default NetworkPolicies | Platform Foundation | P0 |
| PostgreSQL HA cluster | Data Infrastructure | P0 |
| Apache Kafka (KRaft) cluster | Data Infrastructure | P0 |
| Redis cluster | Data Infrastructure | P0 |
| Qdrant deployment | Data Infrastructure | P0 |
| Neo4j community edition | Data Infrastructure | P1 |
| OTEL Collector (DaemonSet) | Observability | P0 |
| Prometheus + Grafana | Observability | P0 |
| Loki (log aggregation) | Observability | P1 |
| Jaeger (tracing) | Observability | P1 |
| Model Plane API (C# / ASP.NET) | Model Plane | P0 |
| Anthropic Claude adapter | Model Plane | P0 |
| OpenAI adapter | Model Plane | P0 |
| Ollama adapter (local models) | Model Plane | P1 |
| Model registry (PostgreSQL) | Model Plane | P0 |
| Rate limiting (Redis) | Model Plane | P0 |
| Security Plane — Auth Service | Security Plane | P0 |
| Security Plane — OPA integration | Security Plane | P0 |
| Basic tenant onboarding (manual) | Multi-tenancy | P1 |
| First service deployed to platform | Integration | P0 |

---

## Success Criteria

- [ ] Kubernetes cluster running and stable (99.9% uptime for 2 weeks)
- [ ] Vault HA cluster auto-unseals and rotates credentials
- [ ] First model invocation through Model Plane returns correct response
- [ ] All model calls appear in OTEL traces (end-to-end)
- [ ] Governance audit event recorded for every model invocation
- [ ] Cost attribution per model call tracked in PostgreSQL
- [ ] First tenant namespace provisioned with full isolation
- [ ] Argo CD deploys all services from Git without manual intervention
- [ ] Zero secrets in any configuration file or code (Vault only)

---

## Dependencies

| Dependency | Type | Owner |
|---|---|---|
| Phase 1 blueprint complete | Prerequisite | Architecture Team |
| Cloud account provisioned (if cloud) | External | Infrastructure |
| Network setup approved by security | External | Security Team |
| Vault Enterprise license (if needed) | External | Procurement |
| AI provider API keys | External | Platform Team |

---

## Estimated Complexity

| Area | Complexity | Effort (person-weeks) |
|---|---|---|
| Kubernetes setup | High | 4 |
| Vault + PKI | High | 3 |
| Data infrastructure | Medium | 4 |
| Observability stack | Medium | 3 |
| Model Plane | High | 6 |
| Security basics | High | 4 |
| Integration testing | Medium | 3 |

**Total Estimated: 27 person-weeks (2 engineers × 14 weeks)**

---

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Vault complexity underestimated | High | Allocate 1 dedicated engineer to Vault |
| Kafka performance at scale | Medium | Load test early (Phase 2 week 8) |
| Provider API rate limits | Low | Rate limiting in Model Plane (Phase 2 deliverable) |
| Network policy misconfiguration | High | Automated network policy testing |
