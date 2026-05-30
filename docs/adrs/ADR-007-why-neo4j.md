# ADR-007 — Why Neo4j for Knowledge Graph

| Field | Value |
|---|---|
| **ADR Number** | 007 |
| **Title** | Neo4j as the Primary Knowledge Graph Database |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

The Knowledge Plane and Semantic Plane require a graph database capable of:
- Storing organizational knowledge as entity-relationship graphs
- Executing complex graph traversal queries for AI context retrieval
- Supporting ontology representation (classes, properties, relationships)
- Enabling knowledge-augmented RAG (GraphRAG)
- Entity resolution across multiple data sources
- Multi-hop reasoning chains for agent decision support

Key requirements:
- Native graph storage (not a graph layer on top of a relational DB)
- Cypher query language (industry standard)
- ACID transactions
- OSS or community edition available
- Enterprise-grade for production workloads
- On-premises deployment support

---

## Decision

**We will use Neo4j Community Edition (with Enterprise Edition considered for production SLAs) as the primary knowledge graph database.**

**Secondary option:** Kuzu for embedded/lightweight graph workloads and development environments.

---

## Alternatives Considered

### Alternative 1 — Amazon Neptune
**Pros:**
- Managed; no operational overhead
- Supports both Gremlin and SPARQL
- High availability built-in

**Cons:**
- AWS-only; violates Cloud Agnostic principle (P4)
- Cannot run on-premises
- Gremlin syntax is less developer-friendly than Cypher
- No OSS version; fully commercial
- Data sovereignty concerns

**Verdict:** Rejected. Violates cloud-agnostic and OSS-first principles.

---

### Alternative 2 — JanusGraph
**Pros:**
- Open source (Apache License)
- Supports multiple storage backends (Cassandra, HBase, BigTable)
- Scales horizontally

**Cons:**
- Significantly more complex to operate (requires Cassandra or HBase)
- Gremlin query language less productive than Cypher
- Smaller community than Neo4j
- Slower query performance on complex traversals vs Neo4j
- Less tooling and ecosystem

**Verdict:** Rejected. Operational complexity is too high relative to benefits.

---

### Alternative 3 — TypeDB (formerly Grakn)
**Pros:**
- Strong type system with polymorphism
- Excellent for ontology-heavy knowledge representation

**Cons:**
- Smaller community
- Query language (TypeQL) is niche and non-standard
- Less tooling
- Uncertain long-term product direction

**Verdict:** Rejected. Non-standard query language and smaller ecosystem.

---

### Alternative 4 — Apache Jena (Triple Store / SPARQL)
**Pros:**
- W3C standards (RDF, SPARQL, OWL)
- Excellent for ontology and semantic web workloads
- Open source

**Cons:**
- SPARQL is complex compared to Cypher
- Not optimized for property graph traversals
- Lower throughput for highly connected graphs
- Smaller AI ecosystem integration

**Verdict:** Considered for semantic reasoning use cases (ontology management) but not primary graph store. Apache Jena may be used in the Semantic Plane.

---

### Alternative 5 — Kuzu
**Pros:**
- Embedded graph database (no separate server)
- Very fast analytical graph queries
- Apache 2.0 license
- Python-native integration

**Cons:**
- Not designed for concurrent write-heavy workloads
- Not suitable as a shared multi-tenant production store
- No clustering support (yet)

**Verdict:** Selected as **secondary** option for: development environments, single-tenant embedded deployments, and analytical graph workloads.

---

## Why Neo4j Wins

| Criterion | Neo4j |
|---|---|
| Graph storage model | Native property graph |
| Query language | Cypher (industry standard, highly readable) |
| ACID compliance | Yes |
| Open Source | Community Edition (GPLv3) |
| Enterprise Edition | Available (production SLA, clustering, security) |
| On-premises | Yes |
| Kubernetes | Yes (official Helm chart) |
| Python driver | First-class |
| Graph algorithm library | Neo4j GDS (Graph Data Science) |
| Ecosystem maturity | 15+ years; largest graph DB community |
| Vector index support | Neo4j 5.x (native vector index for GraphRAG) |
| LangChain integration | Native Neo4j vector store |
| AI ecosystem | Strong: LangChain, LlamaIndex GraphRAG |

---

## GraphRAG Use Case

Neo4j enables a key capability: **GraphRAG** — Retrieval Augmented Generation that traverses the knowledge graph to provide richer, more accurate context to AI models.

```
User Query → Extract Entities → Graph Traversal → 
Collect Related Nodes (depth 2-3) → Augment LLM Context → Response
```

This significantly outperforms pure vector search for structured knowledge retrieval (e.g., "What are the regulatory requirements for derivatives trading in the EU?").

---

## Kuzu Secondary Role

Kuzu is selected for:
- **Development environments:** Embedded, no server setup required
- **Edge deployments:** Single-node, resource-constrained environments
- **Analytical workloads:** Batch graph analytics without online transactional load
- **Local agent memory:** Agent-local knowledge stores that don't require shared access

---

## Consequences

### Positive
- Cypher query language is readable and productive for complex graph queries
- Neo4j GDS enables graph algorithms (PageRank, community detection) for entity importance
- Native vector index in Neo4j 5.x enables hybrid graph+vector search in a single database
- Strong LangChain and LlamaIndex integrations for AI context retrieval

### Negative
- Neo4j Community Edition has no clustering support (single node only)
- Enterprise Edition required for high-availability production deployments (commercial license)
- Graph data modeling requires domain expertise to get right

### Mitigations
- Use Community Edition for development and testing
- Budget for Enterprise Edition in production (or evaluate Apache AGE as OSS alternative)
- Invest in graph data modeling training for the team

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Cypher expressiveness | Enterprise Edition licensing cost |
| Mature ecosystem and tooling | Graph data modeling expertise required |
| GraphRAG capabilities | Single-node limit in Community Edition |
| Native vector index | Operational complexity of graph database |
