# Skill: RAG Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Data Plane, Knowledge Plane, Agent Runtime
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable patterns for building Retrieval-Augmented Generation (RAG) pipelines on the platform. These patterns address the common failure modes of naive RAG implementations in enterprise environments.

---

## RAG Quality Framework

Before choosing a pattern, assess:
1. **Query type:** Factual lookup, analytical reasoning, or synthesis?
2. **Knowledge type:** Structured (graph) or unstructured (documents)?
3. **Latency requirement:** Real-time (< 500ms) or batch (minutes)?
4. **Accuracy requirement:** Can errors be tolerated or are they business-critical?

---

## Pattern 1 — Basic Vector RAG

**When to use:** Simple factual queries against a document corpus. Starting point for all RAG implementations.

**Pipeline:**
```
Query → Embed Query → Qdrant Search (top-K) → Context Assembly → LLM → Response
```

**Configuration starting point:**
```python
from platform_ai.rag import RAGPipeline

rag = RAGPipeline(
    collection=f"tenant_{tenant_id}_documents",
    retrieval_k=5,
    embedding_model="text-embedding-3-small",
    chunk_overlap=50,
)
```

**Common failure mode:** Retrieves semantically similar but contextually wrong chunks.
**Fix:** Add metadata filtering (date range, document type, source system).

---

## Pattern 2 — Hybrid RAG (Dense + Sparse)

**When to use:** Queries containing specific terminology, product codes, or regulatory references that vector search handles poorly.

**Pipeline:**
```
Query → [Vector Search (dense)] + [BM25 Search (sparse)] → RRF Fusion → Rerank → Context → LLM
```

**Reciprocal Rank Fusion (RRF):**
```python
def reciprocal_rank_fusion(vector_results, keyword_results, k=60):
    scores = {}
    for rank, doc in enumerate(vector_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank + 1)
    for rank, doc in enumerate(keyword_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

**When hybrid outperforms pure vector:**
- Regulatory clause retrieval ("Section 4(b) of FCA MCOB 11")
- Product code lookup ("LTV for BTL-2024-Q3")
- Named entity matching ("Barclays Bank plc counterparty exposure")

---

## Pattern 3 — GraphRAG (Knowledge Graph + Vector)

**When to use:** Questions requiring multi-hop reasoning or relationship traversal. "What regulations apply to this product?" or "What is the risk exposure to this counterparty's parent company?"

**Pipeline:**
```
Query → Entity Extraction → Graph Traversal (Neo4j Cypher)
      → Vector Search (semantic context)
      → Merge + Rerank
      → Context Assembly
      → LLM
```

**Entity extraction for graph anchoring:**
```python
async def extract_entities_for_graph(query: str) -> list[Entity]:
    # Use lightweight NER model (not main LLM) to extract anchor entities
    entities = await ner_service.extract(query, entity_types=["Organization", "Product", "Regulation", "Person"])
    
    # Resolve entities to graph canonical IDs
    resolved = await knowledge_service.resolve_entities(entities, tenant_id=tenant_id)
    return resolved

async def graphrag_retrieve(query: str, tenant_id: str):
    entities = await extract_entities_for_graph(query)
    
    # Graph traversal: get N-hop neighborhood of anchor entities
    graph_context = await neo4j_service.get_neighborhood(
        entity_ids=[e.id for e in entities],
        max_depth=2,
        max_nodes=50
    )
    
    # Vector search for semantically related chunks
    vector_context = await qdrant_service.search(
        query_embedding=await embed(query),
        collection=f"tenant_{tenant_id}_documents",
        k=5
    )
    
    return merge_and_rerank(graph_context, vector_context, query)
```

---

## Pattern 4 — Multi-Vector RAG

**When to use:** Long documents (reports, legal contracts) where different embedding representations serve different query types.

**Multiple embeddings per document:**
```
Document → [Summary Embedding] (for high-level queries)
         → [Full Text Embedding] (for detailed queries)
         → [Key Entities Embedding] (for entity-focused queries)

All stored as named vectors in Qdrant:
point.vectors = {
    "summary": embed(document.summary),
    "full_text": embed(document.full_text),
    "key_entities": embed(", ".join(document.key_entities))
}
```

**Query routing:**
```python
def select_vector_type(query: str) -> str:
    if query_classifier.is_summary_query(query):
        return "summary"
    elif query_classifier.is_entity_query(query):
        return "key_entities"
    else:
        return "full_text"
```

---

## Pattern 5 — Reranking Pipeline

**When to use:** After initial retrieval (always recommended for high-stakes RAG).

**Two-stage retrieval:**
```
Stage 1 (Recall): Retrieve top-20 candidates (fast ANN search)
Stage 2 (Precision): Rerank to top-5 (slow cross-encoder)
```

**Cross-encoder reranking:**
```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, candidates: list[Chunk], top_k: int = 5) -> list[Chunk]:
    pairs = [(query, chunk.text) for chunk in candidates]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(scores, candidates), reverse=True)
    return [chunk for _, chunk in ranked[:top_k]]
```

---

## Pattern 6 — Query Expansion

**When to use:** Domain-specific vocabulary where user queries may not match document terminology.

**Ontology-guided expansion:**
```python
async def expand_query(query: str, tenant_id: str) -> list[str]:
    # Extract key terms
    terms = await term_extractor.extract(query)
    
    # Get synonyms and related terms from Semantic Plane
    expanded_terms = []
    for term in terms:
        synonyms = await semantic_service.get_synonyms(term, tenant_id)
        related = await semantic_service.get_related_terms(term, tenant_id)
        expanded_terms.extend(synonyms + related)
    
    # Generate multiple query variants
    variants = [query]
    variants.append(f"{query} {' '.join(expanded_terms[:5])}")
    
    return variants

# Run all query variants in parallel, merge results
```

---

## Chunking Strategy Guide

| Document Type | Strategy | Chunk Size | Overlap |
|---|---|---|---|
| Policy documents | Semantic (sentence boundary) | 512 tokens | 50 tokens |
| Legal contracts | Section-based chunking | Variable (section = chunk) | 0 |
| Financial reports | Paragraph + table extraction | 256 tokens | 25 tokens |
| Regulatory text | Article/paragraph-based | Variable | 0 |
| Emails | Conversation-based | Full email | 0 |
| Technical specs | Function/section-based | 256 tokens | 0 |

---

## RAG Evaluation Checklist

Before deploying a RAG pipeline to production:

- [ ] Retrieval quality tested on golden dataset (Context Recall > 0.85)
- [ ] Faithfulness tested (Faithfulness > 0.90)
- [ ] Latency acceptable (P95 < 500ms for retrieval, < 3s end-to-end)
- [ ] PII masking verified (no PII in embeddings)
- [ ] Tenant isolation verified (cross-tenant retrieval impossible)
- [ ] Lineage tracking verified (source documents traced in audit)
- [ ] Staleness monitoring configured (alert on knowledge age)
- [ ] Reranking in place for production quality
- [ ] Evaluation dataset registered in Registry Plane
- [ ] RAGAS evaluation in Evaluation Plane passing

---

## Common RAG Failures and Fixes

| Failure | Root Cause | Fix |
|---|---|---|
| Hallucination despite RAG | Retrieved context not used (faithfulness < 0.5) | Strengthen system prompt: "Only answer based on provided context" |
| Wrong chunks retrieved | Chunking destroys context | Use semantic chunking; increase overlap |
| Slow retrieval | Large collection without quantization | Enable Qdrant scalar quantization |
| Missing domain terminology | Generic embedding model | Fine-tune or use domain-specific embedding model |
| Stale knowledge in answers | Knowledge base not updated | Configure freshness alerts; automate re-ingestion |
| PII in retrieved context | PII masking not applied | Verify PII masking before embedding |
