# Phase 6 — Developer Experience

> **Phase:** 6 of 8
> **Status:** Planned
> **Timeline:** Q2-Q3 2027
> **Owner:** Platform Developer Relations Team

---

## Goals

1. Launch the Developer Portal with full API documentation and playground
2. Release Python SDK, C# SDK, and TypeScript SDK
3. Publish the Platform CLI (platform-cli)
4. Create and validate starter templates (RAG agent, multi-agent, workflow)
5. Build the local development environment (Docker Compose stack)
6. Achieve: first external team builds and deploys an agent independently

---

## Deliverables

| Deliverable | Priority |
|---|---|
| Developer Portal (Next.js) — API catalog, docs, getting started | P0 |
| Python SDK (v1.0) | P0 |
| C# SDK (v1.0) | P0 |
| TypeScript SDK (v1.0) | P1 |
| Platform CLI (platform-cli) — deploy, status, eval, audit | P0 |
| Local dev environment (docker-compose.dev.yml) | P0 |
| API Playground (interactive testing in portal) | P1 |
| Starter template: RAG Agent | P0 |
| Starter template: Multi-Agent Supervisor-Worker | P1 |
| Starter template: Workflow with HITL | P1 |
| Starter template: MCP Server | P1 |
| Onboarding journey (Day 1-3 guided experience) | P0 |
| SDK documentation (docstrings → generated docs) | P0 |
| Video tutorials (3 core use cases) | P1 |
| Platform changelog and migration guides | P0 |
| MCP server for Claude Code (platform as tool) | P1 |
| External team pilot (3 teams) | Milestone |

---

## Success Criteria

- [ ] New AI engineer goes from zero to running first agent in < 30 minutes (local dev)
- [ ] 3 external teams deploy production agents using templates without architecture team help
- [ ] Python SDK covers 100% of Agent Runtime and Model Plane APIs
- [ ] Platform CLI enables complete deployment workflow from terminal
- [ ] API Playground allows testing any endpoint without writing code
- [ ] Local dev environment starts in < 3 minutes (`platform dev start`)
- [ ] SDK documentation complete (no undocumented public methods)
- [ ] At least 2 template-based agents in production from external teams

---

## Developer Onboarding Target Journey

```
Day 0:  Account created, access granted, API keys provisioned
Day 1:  Clone template → Run locally → First agent responds in 30 min
Day 2:  Connect knowledge base → RAG working → Evaluation passing
Day 3:  Deploy to staging → Governance audit visible → HITL tested
Week 2: Production deployment → Monitoring dashboard → First real use case
```

---

## Dependencies

| Dependency | Notes |
|---|---|
| Phases 2-5 complete | SDK needs working APIs to wrap |
| External pilot teams | Recruit 3 willing teams early |
| Technical writer | SDK docs and portal content |
| Design (UX) | Developer portal UX |

---

## Estimated Complexity

| Area | Complexity | Effort (person-weeks) |
|---|---|---|
| Developer Portal | High | 8 |
| Python SDK | Medium | 5 |
| C# SDK | Medium | 4 |
| Platform CLI | Medium | 5 |
| Local dev environment | Medium | 3 |
| Templates | Medium | 4 |
| Documentation | High | 6 |
| Pilot team support | Medium | 4 |

**Total Estimated: 39 person-weeks (3 engineers × 13 weeks)**
