# ADR-004 — Why Kubernetes for Container Orchestration

| Field | Value |
|---|---|
| **ADR Number** | 004 |
| **Title** | Kubernetes as the Container Orchestration Platform |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team, Infrastructure Team |
| **Supersedes** | None |

---

## Context

The AI Operating Platform is composed of many services — C# platform services, Python AI runtimes, Kafka brokers, PostgreSQL, Redis, Qdrant, Neo4j, and observability components. These services must be:

- Deployed consistently across cloud providers and on-premises
- Scaled horizontally based on load
- Isolated per tenant
- Updated without downtime
- Monitored and healed automatically
- Networked securely with policy enforcement

The question is: what container orchestration platform to use?

---

## Decision

**We will use Kubernetes as the container orchestration platform.**

Kubernetes is deployed using:
- **Cloud:** Managed Kubernetes (EKS on AWS, AKS on Azure, GKE on GCP)
- **On-premises:** RKE2 (Rancher Kubernetes Engine 2) or k3s for edge/lightweight deployments

All platform components are Kubernetes-native. No component depends on a specific Kubernetes distribution.

---

## Alternatives Considered

### Alternative 1 — Docker Swarm
**Pros:**
- Simpler to operate
- Integrated with Docker CLI

**Cons:**
- Significantly less powerful than Kubernetes
- No support for advanced networking policies
- No RBAC for multi-tenancy
- Limited ecosystem (no Helm, limited operators)
- Industry adoption declining

**Verdict:** Rejected. Insufficient for enterprise AI platform requirements.

---

### Alternative 2 — AWS ECS / Fargate
**Pros:**
- Managed by AWS; no control plane to operate
- Integrated with AWS services (IAM, CloudWatch, ALB)

**Cons:**
- AWS-only; violates Cloud Agnostic principle (P4)
- Cannot run on-premises
- No Kubernetes ecosystem (no Helm, no operators)
- Cannot use Kubernetes NetworkPolicies for tenant isolation

**Verdict:** Rejected. Violates cloud-agnostic principle.

---

### Alternative 3 — HashiCorp Nomad
**Pros:**
- Simpler than Kubernetes
- Works well with Vault and Consul (same HashiCorp ecosystem)
- Supports Docker, VMs, and raw binaries

**Cons:**
- Much smaller ecosystem than Kubernetes
- Fewer operators and community support for AI workloads
- Less mature multi-tenancy support
- Platform engineers less familiar with Nomad

**Verdict:** Rejected. Kubernetes ecosystem advantage is decisive.

---

## Why Kubernetes

| Capability | Kubernetes Support |
|---|---|
| Multi-tenancy (namespace isolation) | Native |
| Network policy enforcement | Native (CNI plugins) |
| RBAC | Native |
| Horizontal pod autoscaling | Native |
| Rolling updates / rollbacks | Native |
| Secret management integration | Vault Agent Injector |
| Service mesh | Istio / Linkerd (CSI) |
| GPU workload support | Native (for local model inference) |
| Operator pattern for stateful services | Rich ecosystem |
| Helm package management | Standard |
| Multi-cloud portability | Industry standard |
| GitOps (Argo CD, Flux) | First-class support |

---

## Multi-Tenancy Model in Kubernetes

The platform uses a **namespace-per-tenant** model:

```
cluster/
├── namespace: platform-system       (platform infrastructure)
├── namespace: tenant-bankA          (Bank A workloads)
├── namespace: tenant-bankB          (Bank B workloads)
├── namespace: tenant-insurerC       (Insurer C workloads)
└── namespace: observability         (shared observability stack)
```

Each tenant namespace has:
- ResourceQuota (CPU, memory limits)
- LimitRange (default container limits)
- NetworkPolicy (no cross-namespace traffic by default)
- ServiceAccount per service
- RBAC Role + RoleBinding (tenant engineers only see their namespace)

---

## Consequences

### Positive
- Cloud-agnostic: same Kubernetes manifests deploy to AWS, Azure, GCP, on-premises
- Rich ecosystem: Helm, Argo CD, Istio, cert-manager, Vault Agent Injector
- Strong multi-tenancy via namespaces + NetworkPolicies
- GPU support for local model inference (NVIDIA device plugin)
- Battle-tested at enterprise scale
- Strong security tooling (OPA/Gatekeeper, Kyverno for policy enforcement)

### Negative
- Kubernetes has high operational complexity
- Control plane management required for self-hosted deployments
- Steep learning curve for teams new to Kubernetes

### Mitigations
- Use managed Kubernetes on cloud (EKS/AKS/GKE) to offload control plane
- RKE2 for on-premises (production-grade, CNCF certified)
- Platform team owns Kubernetes operations; application teams use namespace abstractions

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Cloud-agnostic deployment | Kubernetes operational complexity |
| Rich ecosystem and operators | Control plane management overhead |
| Strong multi-tenancy | Namespace management at scale |
| Industry standard (talent available) | Learning curve for beginners |
