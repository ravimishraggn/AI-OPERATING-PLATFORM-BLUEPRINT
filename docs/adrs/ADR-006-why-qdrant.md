# ADR-006 — Why Qdrant for Vector Search

| Field | Value |
|---|---|
| **ADR Number** | 006 |
| **Title** | Qdrant as the Vector Database for Semantic Search and RAG |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

The AI Operating Platform requires a vector database for:
- Semantic document search in RAG pipelines
- Agent memory storage (episodic and semantic memory)
- Knowledge retrieval for context augmentation
- Similarity search across embedded entities
- Multi-tenant vector collections (each tenant's knowledge isolated)

Key requirements:
- **Performance:** Sub-100ms similarity search at millions of vectors
- **Filtering:** Metadata-based filtering combined with vector similarity
- **Multi-tenancy:** Isolated collections or payload-based filtering per tenant
- **Open Source:** Must be self-hostable
- **Scalability:** Horizontal scaling for high-volume RAG workloads
- **On-premises:** Must deploy to air-gapped environments

---

## Decision

**We will use Qdrant as the vector database.**

---

## Alternatives Considered

### Alternative 1 — Pinecone
**Pros:**
- Fully managed; no operational overhead
- Excellent performance
- Simple API

**Cons:**
- Cloud-only SaaS; cannot run on-premises
- Violates Cloud Agnostic principle (P4)
- Violates Open Source First principle (P2)
- Data sovereignty: vectors stored in Pinecone's infrastructure
- Expensive at enterprise scale
- No self-hosted option

**Verdict:** Rejected. Violates OSS first and cloud-agnostic principles.

---

### Alternative 2 — Weaviate
**Pros:**
- Open source
- Built-in vectorization modules
- GraphQL interface
- Good multi-tenancy support

**Cons:**
- Higher memory footprint than Qdrant
- Vectorization modules can be slower than standalone embedding services
- More complex configuration
- Performance benchmarks slightly below Qdrant for pure vector search

**Verdict:** Strong alternative. Not selected due to Qdrant's superior raw performance and simpler Rust-based architecture.

---

### Alternative 3 — Milvus
**Pros:**
- Open source
- Excellent scalability (designed for billion-scale vectors)
- GPU acceleration support

**Cons:**
- Significantly more complex to operate (etcd, MinIO dependencies)
- Heavier resource footprint
- Overkill for most RAG workloads
- Python client less mature than Qdrant's

**Verdict:** Rejected for primary use. Milvus is considered for future use at extreme scale.

---

### Alternative 4 — pgvector (PostgreSQL extension)
**Pros:**
- Already in the stack (PostgreSQL)
- ACID transactions on vectors
- Familiar SQL interface
- No additional service to operate

**Cons:**
- Performance significantly below dedicated vector databases at scale
- Approximate nearest neighbor (ANN) indexing less efficient than HNSW/IVF
- Not designed for billion-scale vectors
- Filtering + vector search performance degrades with scale

**Verdict:** Rejected as primary vector store. pgvector may be used for low-volume or prototype workloads.

---

### Alternative 5 — Chroma
**Pros:**
- Simple API; popular in LangChain ecosystem
- Easy to get started
- Embedded and client-server modes

**Cons:**
- Less production-hardened than Qdrant
- Smaller community
- Multi-tenancy less robust
- Not designed for enterprise scale

**Verdict:** Rejected for production platform. Suitable for development and prototyping.

---

## Why Qdrant Wins

| Criterion | Qdrant |
|---|---|
| Written in | Rust (memory-safe, high performance) |
| Open Source | Yes (Apache 2.0) |
| Self-hosted | Yes (Docker, Kubernetes) |
| On-premises | Yes |
| Multi-tenancy | Collection-per-tenant or payload filtering |
| Filtering | Rich payload filtering with vector search |
| Performance | Top-tier ANN performance (HNSW) |
| Quantization | Scalar and product quantization (memory reduction) |
| Streaming ingestion | Supported |
| Named vectors | Multiple vectors per point (text + image, etc.) |
| Snapshots | Native backup/restore |
| Horizontal scaling | Distributed mode |
| gRPC + REST API | Both available |

---

## Multi-Tenant Strategy

Two supported patterns:

**Pattern 1 — Collection per Tenant** (strong isolation, recommended for regulated industries)
```
tenant_bankA_documents
tenant_bankA_agent_memory
tenant_bankB_documents
tenant_bankB_agent_memory
```

**Pattern 2 — Payload Filtering** (shared collection with tenant_id filter)
```
collection: platform_documents
point.payload: { "tenant_id": "bankA", ... }
filter: { "must": [{ "key": "tenant_id", "match": { "value": "bankA" } }] }
```

Pattern 1 is preferred for regulated industries. Pattern 2 is acceptable for SaaS-style deployments where cost efficiency is prioritized.

---

## Consequences

### Positive
- High-performance vector search enables real-time RAG
- Rust-based implementation: excellent memory safety and performance
- Rich filtering enables hybrid search (semantic + structured criteria)
- Named vectors enable multi-modal search (text + image)
- Collection snapshots enable backup and restore

### Negative
- Additional service to operate and monitor
- Not as simple as pgvector for teams already using PostgreSQL
- Collection management at scale requires automation

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Top-tier vector search performance | Additional service to operate |
| Rich filtering capabilities | Collection management complexity at scale |
| OSS + self-hosted | Not as simple as pgvector |
| Rust-based memory safety | Rust expertise for custom builds |
