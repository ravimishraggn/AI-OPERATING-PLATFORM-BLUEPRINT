# Phase 4 — Agent Runtime

> **Phase:** 4 of 8
> **Status:** Planned
> **Timeline:** Q4 2026 - Q1 2027
> **Owner:** AI Engineering Team

---

## Goals

1. Deploy the Agent Runtime Plane with LangGraph orchestration
2. Implement Kafka-backed agent state checkpointing
3. Build human-in-the-loop (HITL) infrastructure
4. Launch MCP tool registry and first platform MCP tools
5. Implement multi-agent supervisor-worker pattern
6. Deploy the Decision Plane with explainability
7. Launch Workflow Orchestration for multi-step business processes
8. Complete the Evaluation Plane for agent quality measurement
9. Achieve first governed agent completing a real business task

---

## Deliverables

| Deliverable | Priority |
|---|---|
| Agent Registry (C# API + PostgreSQL) | P0 |
| LangGraph Executor with Platform extensions | P0 |
| Kafka checkpointer for agent state | P0 |
| Agent Governance Middleware (pre/post step) | P0 |
| HITL Manager + Kafka queue | P0 |
| MCP Tool Registry | P0 |
| Platform MCP Servers: knowledge-graph, vector-search, document-retrieval | P0 |
| Agent Memory: working (Redis) + episodic (Qdrant) | P1 |
| Multi-agent Supervisor-Worker coordinator | P1 |
| Streaming agent progress (SSE) | P1 |
| Decision Plane API + Decision Record Store | P1 |
| LLM-generated explainability | P1 |
| Workflow Engine (Elsa Workflows) | P1 |
| Human Task Manager + notification | P1 |
| Evaluation Plane: LLM-as-judge | P0 |
| Evaluation Plane: RAGAS evaluation | P0 |
| Quality Gate for agent promotion | P0 |
| Agent sandbox (subprocess isolation) | P1 |
| First governed agent: reference implementation | P0 |

---

## Reference Agent Implementation

Phase 4 delivers a reference agent ("Document Analysis Agent") that demonstrates:
- LangGraph execution with governance middleware
- RAG retrieval for context
- MCP tool calls (knowledge-graph + vector-search)
- HITL gate for low-confidence decisions
- Full audit trail in Governance Plane
- OTEL traces for every step
- Evaluation against golden dataset (quality gate passing)

This reference agent is the template for all future agent development.

---

## Success Criteria

- [ ] Reference agent completes document analysis task end-to-end
- [ ] Every agent step appears in Jaeger distributed trace
- [ ] Agent state survives pod restart and resumes from checkpoint
- [ ] HITL request delivered to reviewer within 5 seconds of trigger
- [ ] Agent cannot call unauthorized MCP tools (authorization enforced)
- [ ] Decision record created for every AI-influenced decision
- [ ] Evaluation quality gate blocks promotion when threshold not met
- [ ] Agent token budget enforced (run terminates at limit)
- [ ] Multi-agent workflow completes with supervisor coordinating 2 workers

---

## Dependencies

| Dependency | Notes |
|---|---|
| Phase 2 Platform Foundation | Model Plane, Kafka, Redis, Vault |
| Phase 3 Knowledge Platform | RAG pipeline, knowledge graph for context |
| LangGraph stability | Pin to specific version in pyproject.toml |
| MCP specification | Pin to specific MCP spec version |

---

## Estimated Complexity

| Area | Complexity | Effort (person-weeks) |
|---|---|---|
| Agent Runtime + LangGraph | Very High | 8 |
| Kafka checkpointer | High | 4 |
| HITL infrastructure | High | 4 |
| MCP tools + registry | High | 5 |
| Decision Plane | Medium | 4 |
| Workflow Engine | High | 5 |
| Evaluation Plane | High | 6 |
| Reference agent | Medium | 3 |

**Total Estimated: 39 person-weeks (3 engineers × 13 weeks)**

---

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| LangGraph API changes | High | Pin version; update in controlled upgrade cycle |
| Kafka checkpoint performance | Medium | Load test with realistic agent workloads |
| HITL UX friction | Medium | Early UX testing with real reviewers |
| MCP protocol immaturity | Medium | Implement adapter layer over MCP client |
