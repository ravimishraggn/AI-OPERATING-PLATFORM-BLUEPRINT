# ADR-005 — Why Apache Kafka for Messaging

| Field | Value |
|---|---|
| **ADR Number** | 005 |
| **Title** | Apache Kafka as the Platform Event Streaming Backbone |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

The AI Operating Platform requires a messaging and event streaming backbone for:
- AI agent event publishing (agent started, step completed, decision made)
- Governance audit event streaming (immutable, durable audit log)
- Data ingestion pipelines (documents, entity changes, knowledge updates)
- Evaluation pipeline triggers (model output published → evaluation consumed)
- Cross-service async communication (decoupled service boundaries)
- Real-time observability event flow (metrics, logs routing)
- Multi-tenant event isolation

Key requirements:
- **Durability:** Events must not be lost; replay must be possible
- **High throughput:** Platform must handle thousands of AI operations per second
- **Ordering guarantees:** Audit events must be ordered
- **Retention:** Governance events retained for 7+ years
- **Multi-tenancy:** Tenant events must be isolated
- **Replay:** Ability to replay events for evaluation or debugging

---

## Decision

**We will use Apache Kafka as the platform event streaming backbone.**

Deployment options:
- **Self-hosted:** Apache Kafka on Kubernetes (KRaft mode, no ZooKeeper)
- **Managed:** Confluent Cloud, MSK (AWS), Event Hubs (Azure Kafka-compatible) — behind an adapter
- **On-premises:** Apache Kafka KRaft on bare-metal or VM

---

## Alternatives Considered

### Alternative 1 — RabbitMQ
**Pros:**
- Simpler to operate than Kafka
- Good for task queues and work distribution
- Strong routing capabilities (exchanges, bindings)

**Cons:**
- Not designed for event streaming (replay not native)
- Message retention is ephemeral by design
- Lower throughput than Kafka at scale
- Not suitable for 7-year audit log retention
- Ordering guarantees weaker than Kafka partitions

**Verdict:** Rejected. Insufficient for durable event streaming and audit requirements.

---

### Alternative 2 — NATS / NATS JetStream
**Pros:**
- Extremely lightweight and fast
- Low operational overhead
- JetStream adds persistence and replay

**Cons:**
- Smaller ecosystem than Kafka
- Less proven at extreme scale
- Fewer enterprise integrations
- Less familiar to enterprise operations teams

**Verdict:** Rejected. Kafka's ecosystem and enterprise track record are decisive.

---

### Alternative 3 — AWS SQS + SNS
**Pros:**
- Fully managed
- Simple to use
- Low operational overhead

**Cons:**
- AWS-only; violates Cloud Agnostic principle (P4)
- Not replayable (messages deleted after consumption)
- 14-day maximum retention
- Cannot run on-premises
- Not suitable for audit log requirements

**Verdict:** Rejected. Violates cloud-agnostic principle.

---

### Alternative 4 — Azure Service Bus
**Pros:**
- Managed service
- Good ordering support

**Cons:**
- Azure-only; violates Cloud Agnostic principle (P4)
- Not suitable for high-throughput event streaming
- Limited retention

**Verdict:** Rejected. Violates cloud-agnostic principle.

---

### Alternative 5 — Redis Streams
**Pros:**
- Already in the stack (Redis for cache)
- Simple API
- Low latency

**Cons:**
- Not designed as a primary event backbone
- Limited consumer group semantics
- No long-term retention capability
- Not suitable for durable audit logs

**Verdict:** Rejected as primary backbone. Redis Streams may be used for short-lived real-time event patterns.

---

## Why Kafka Wins

| Requirement | Kafka |
|---|---|
| Durable event log | Native (log-based storage) |
| Long-term retention (7+ years) | Configurable per topic |
| High throughput (millions of events/day) | Proven at petabyte scale |
| Event replay | Native |
| Consumer groups (multiple services consume same event) | Native |
| Ordering guarantees (per partition) | Native |
| Multi-tenancy (topic per tenant or topic naming convention) | Supported |
| On-premises deployment | Native |
| Cloud-managed option | MSK, Confluent, Event Hubs |
| Ecosystem (Kafka Connect, Kafka Streams) | Rich |

---

## Topic Structure

```
platform.{tenant_id}.agent.events       ← Agent lifecycle events
platform.{tenant_id}.governance.audit   ← Governance audit events (long retention)
platform.{tenant_id}.data.ingestion     ← Data pipeline events
platform.{tenant_id}.evaluation.results ← Evaluation pipeline output
platform.{tenant_id}.knowledge.updates  ← Knowledge graph change events
platform.system.metrics                 ← Platform-level metrics
platform.system.alerts                  ← Platform alerts
```

---

## Consequences

### Positive
- Durable, replayable event log enables audit, debugging, and replay-based evaluation
- High throughput handles enterprise AI operation volumes
- Kafka Connect enables data source integration without custom code
- KRaft mode (Kafka without ZooKeeper) simplifies deployment
- Topic-per-tenant or naming convention enables tenant isolation

### Negative
- Higher operational complexity than simpler queues
- Requires tuning (partition counts, replication factor, retention)
- Storage requirements for long-retention topics
- Consumer lag monitoring required

### Mitigations
- Use managed Kafka (Confluent Cloud or MSK) for cloud deployments
- KRaft mode reduces operational complexity vs ZooKeeper
- Strimzi Kafka Operator for Kubernetes-native management

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Durable, replayable event log | Higher operational complexity |
| 7-year audit retention | Storage cost for long-retention topics |
| High throughput | Partition management overhead |
| Replay for debugging/evaluation | Consumer lag monitoring |
