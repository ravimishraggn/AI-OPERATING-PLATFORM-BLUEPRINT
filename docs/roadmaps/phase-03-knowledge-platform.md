# Phase 3 — Knowledge Platform

> **Phase:** 3 of 8
> **Status:** Planned
> **Timeline:** Q3-Q4 2026
> **Owner:** Knowledge Engineering Team

---

## Goals

1. Deploy the Data Plane with full ingestion pipeline (batch and streaming)
2. Implement PII detection, masking, and data sovereignty controls
3. Deploy the Knowledge Plane with Neo4j knowledge graph
4. Implement the Semantic Plane with core platform ontology
5. Build and validate a complete RAG pipeline (hybrid vector + keyword + graph)
6. Achieve first accurate GraphRAG query against real organizational knowledge

---

## Deliverables

| Deliverable | Priority |
|---|---|
| Data ingestion pipeline (Kafka Connect + batch) | P0 |
| Document chunking service (multiple strategies) | P0 |
| Embedding generation service (Sentence Transformers) | P0 |
| Qdrant vector population and search | P0 |
| PII Detection Service (Presidio integration) | P0 |
| Data masking service | P0 |
| Data lineage tracking (OpenLineage) | P0 |
| Data sovereignty enforcement (movement check) | P0 |
| Entity extraction pipeline (GLiNER + spaCy) | P1 |
| Entity resolution service | P1 |
| Neo4j knowledge graph population | P1 |
| Cypher query service for AI context retrieval | P1 |
| Knowledge freshness monitor | P1 |
| Core Platform Ontology (formal OWL representation) | P1 |
| Semantic Plane API | P1 |
| Basic RAG pipeline (vector only) | P0 |
| Hybrid RAG (vector + BM25) | P1 |
| GraphRAG (vector + graph traversal) | P1 |
| Reranking pipeline | P1 |
| RAGAS evaluation for RAG quality | P0 |
| First tenant knowledge base populated | Milestone |

---

## Success Criteria

- [ ] 10,000 documents ingested through pipeline end-to-end
- [ ] PII correctly detected and masked (< 1% false negative on test set)
- [ ] RAG query returns accurate answer (faithfulness > 0.90 on golden dataset)
- [ ] GraphRAG traverses knowledge graph for multi-hop queries
- [ ] Data lineage traceable from source document to RAG result
- [ ] Sovereignty check blocks cross-region data transfer
- [ ] Entity extraction identifies key entity types (org, person, regulation, product)
- [ ] Knowledge graph contains > 1,000 entities with relationships

---

## Dependencies

| Dependency | Notes |
|---|---|
| Phase 2 Platform Foundation | Qdrant, Neo4j, PostgreSQL, Kafka running |
| Sample document corpus | Required for testing and golden dataset |
| Domain expert | For entity extraction validation |
| Sovereignty rules from Legal | Required for data movement policies |

---

## Estimated Complexity

| Area | Complexity | Effort (person-weeks) |
|---|---|---|
| Data ingestion pipeline | High | 5 |
| PII detection + masking | Medium | 3 |
| Data sovereignty | High | 4 |
| Knowledge graph pipeline | Very High | 8 |
| RAG pipeline | High | 6 |
| Semantic Plane | Medium | 3 |
| Evaluation (RAGAS) | Medium | 2 |

**Total Estimated: 31 person-weeks (2-3 engineers × 12 weeks)**

---

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Entity extraction accuracy insufficient | High | Use ensemble (GLiNER + GPT extraction); human validation for critical entities |
| RAG quality below threshold | High | Iterative improvement with RAGAS; reranker significantly improves precision |
| Knowledge graph scalability | Medium | Benchmark Neo4j at 1M entity scale in Phase 3 |
| Embedding model selection | Medium | Benchmark 3-5 models on domain-specific test set before committing |
