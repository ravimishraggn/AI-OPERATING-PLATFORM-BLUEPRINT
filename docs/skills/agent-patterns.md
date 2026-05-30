# Skill: Agent Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Agent Runtime
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

This document provides reusable architectural patterns for building AI agents on the platform. These patterns are battle-tested in regulated enterprise environments and should be the starting point for all agent design work.

---

## Pattern 1 — ReAct Agent (Standard Pattern)

**When to use:** Single-domain tasks requiring tool use and reasoning.

**Structure:**
```
Input → [Think → Act → Observe] loop → Output
```

**LangGraph Implementation:**
```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode

def should_continue(state):
    messages = state["messages"]
    last_message = messages[-1]
    if last_message.tool_calls:
        return "tools"
    return END

graph = StateGraph(AgentState)
graph.add_node("agent", call_model)
graph.add_node("tools", ToolNode(tools))
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")
```

**Governance hooks:**
- Policy check before each "agent" node (is this allowed to run?)
- Tool call authorization before each tool invocation
- All steps checkpointed to Kafka

---

## Pattern 2 — Plan-Execute-Review

**When to use:** Multi-step tasks with unknown number of steps; complex research or analysis.

**Structure:**
```
Input → Planner → [Execute step → Review] × N → Synthesize → Output
```

**Key design decisions:**
- Planner runs once with full context
- Each executor step is isolated (can be run in parallel)
- Reviewer checks step quality before moving on
- Re-planning triggered if executor consistently fails

**HITL gate:** Add interrupt before Synthesize for human quality check on high-stakes tasks.

---

## Pattern 3 — Supervisor-Worker (Multi-Agent)

**When to use:** Tasks requiring multiple specialized perspectives; cross-domain analysis.

**Structure:**
```
Input → Supervisor
           ├── Worker A (domain expert)
           ├── Worker B (fact checker)
           └── Worker C (regulatory expert)
        → Supervisor (synthesize + adjudicate)
        → Output
```

**Critical design rules:**
- Each worker has its own capability scope (not the supervisor's scope)
- Workers cannot communicate directly (all coordination through supervisor)
- Worker failure → supervisor decides whether to retry, escalate, or proceed without
- Governance profile: each worker must be compatible with supervisor's profile

**Authorization model:**
```python
# Supervisor cannot grant workers permissions it doesn't have
supervisor_tools = {"knowledge-graph", "vector-search"}
worker_max_tools = supervisor_tools.intersection(worker_declared_tools)
# Worker can only use tools that supervisor is also authorized for
```

---

## Pattern 4 — Human-in-the-Loop (HITL)

**When to use:** High-stakes decisions; confidence below threshold; regulatory requirement.

**HITL trigger conditions (configure per agent):**
```yaml
hitl_conditions:
  - decision_type: "approve_or_decline"
    confidence_threshold: 0.85
    # Required if confidence below threshold
  - outcome: "DECLINE"
    always_require: true
    # Always require human review for declines
  - data_classification: "restricted"
    always_require: true
    # Always require human for restricted data decisions
```

**HITL suspension/resume pattern:**
```python
# Agent graph node - will suspend here
@graph.node
async def request_human_review(state: AgentState) -> Command:
    # Post HITL request to Kafka
    await hitl_service.create_request(
        run_id=state["run_id"],
        question=state["decision"]["question"],
        context=state["decision"]["context"],
        options=state["decision"]["options"],
        deadline=datetime.utcnow() + timedelta(hours=24)
    )
    # Return Command to interrupt (suspend graph)
    return Command(interrupt=True)

# Resume: platform injects human decision into state
# Graph continues from this node with human_decision populated
```

---

## Pattern 5 — Agentic RAG (Iterative Retrieval)

**When to use:** Complex queries requiring multiple retrieval steps; multi-hop reasoning.

**Structure:**
```
Query → Initial Retrieval → Evaluate Sufficiency
                              ├── Sufficient → Generate Answer
                              └── Insufficient → Reformulate Query → Retrieve Again
                                                                    → (max 3 iterations)
```

**Termination conditions (prevent infinite loops):**
- Max 3 retrieval iterations
- Sufficiency score > 0.75 (LLM-as-judge)
- No new information found in last retrieval

---

## Pattern 6 — Streaming Agent with Live Progress

**When to use:** Long-running agents where user needs to see progress in real-time.

**Implementation:**
```python
# API endpoint
@router.post("/agents/{agent_id}/runs/stream")
async def stream_agent_run(agent_id: str, request: AgentRunRequest):
    async def event_generator():
        async for event in agent_executor.stream(request.input):
            if event["type"] == "on_chat_model_stream":
                yield f"data: {json.dumps({'type': 'token', 'content': event['data']})}\n\n"
            elif event["type"] == "on_tool_start":
                yield f"data: {json.dumps({'type': 'tool_start', 'tool': event['name']})}\n\n"
            elif event["type"] == "on_tool_end":
                yield f"data: {json.dumps({'type': 'tool_end', 'result': event['data']})}\n\n"
    
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

---

## Pattern 7 — Critic-Revise

**When to use:** Tasks requiring high-quality outputs; document generation; analysis reports.

**Structure:**
```
Input → Generator Agent → Critic Agent → Revise if needed → Output
```

**Critic prompt pattern:**
```
You are a critical reviewer. Evaluate the following output for:
1. Factual accuracy (is everything grounded in provided context?)
2. Completeness (does it address all aspects of the question?)
3. Regulatory compliance (are all claims within legal bounds?)
4. Clarity (is it understandable to the target audience?)

Provide a score 0-10 and specific issues to address.
If score < 7, list required revisions.
```

**Max revision cycles:** 2 (prevent infinite revision loops).

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Unbounded tool loops | Agent loops forever calling tools | Max iteration count + termination condition |
| Direct provider SDK calls | Bypasses governance, audit, cost tracking | Always use Model Plane |
| Hardcoded tool endpoints | Tight coupling, no authorization | Use MCP Tool Registry |
| Agent calling other agents directly | No supervisor oversight | Supervisor-worker pattern with approval |
| Silent exception swallowing | No audit trail for failures | All errors logged + HITL escalation |
| Storing PII in agent state | Privacy violation | Reference IDs only; retrieve from governed store |

---

## Token Budget Management

Every agent must declare and respect its token budget:

```python
class AgentBudget:
    max_input_tokens: int = 100_000
    max_output_tokens: int = 10_000
    max_tool_calls: int = 20
    max_runtime_seconds: int = 300

# Before each model call, check budget
remaining = budget.max_input_tokens - state["tokens_consumed"]
if remaining < min_tokens_for_response:
    # Trigger HITL or graceful termination
    return summarize_and_stop(state)
```

---

## Observability Standards for Agents

Every agent must emit these OpenTelemetry attributes:

```python
# Required span attributes for every agent step
span.set_attribute("ai.agent.id", agent_id)
span.set_attribute("ai.agent.run_id", run_id)
span.set_attribute("ai.agent.step_number", step_number)
span.set_attribute("ai.agent.step_name", step_name)
span.set_attribute("ai.agent.tokens_consumed", tokens_consumed)
span.set_attribute("ai.tenant.id", tenant_id)
span.set_attribute("ai.model.id", model_id)
span.set_attribute("ai.model.provider", provider)
```
