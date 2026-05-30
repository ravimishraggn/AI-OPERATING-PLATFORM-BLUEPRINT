# Skill: Observability Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Observability Plane
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable patterns for instrumenting platform services and AI operations with OpenTelemetry. All platform services must follow these patterns for consistent observability.

---

## Pattern 1 — OTEL Initialization (Service Bootstrap)

### Python (FastAPI)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

def configure_telemetry(service_name: str, tenant_id: str):
    provider = TracerProvider(
        resource=Resource.create({
            "service.name": service_name,
            "service.version": os.getenv("SERVICE_VERSION", "unknown"),
            "deployment.environment": os.getenv("ENVIRONMENT", "development"),
            "tenant.id": tenant_id,
        })
    )
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
    )
    trace.set_tracer_provider(provider)
    
    # Auto-instrument FastAPI and HTTPX
    FastAPIInstrumentor.instrument_app(app)
    HTTPXClientInstrumentor().instrument()
```

### C# (ASP.NET Core)

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: "platform-governance-api")
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = Environment.GetEnvironmentVariable("ENVIRONMENT") ?? "development",
        }))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddOtlpExporter(opts => opts.Endpoint = new Uri("http://otel-collector:4317")))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());
```

---

## Pattern 2 — AI Operation Span

**Every AI model invocation must create a span with these attributes:**

```python
tracer = trace.get_tracer("platform.model-plane")

async def invoke_model_with_tracing(request: ModelRequest) -> ModelResponse:
    with tracer.start_as_current_span("model.invoke") as span:
        span.set_attribute("ai.model.provider", request.provider)
        span.set_attribute("ai.model.id", request.model_id)
        span.set_attribute("ai.tenant.id", request.tenant_id)
        span.set_attribute("ai.agent.id", request.agent_id or "direct")
        span.set_attribute("ai.use_case", request.use_case)
        span.set_attribute("ai.model.temperature", request.temperature)
        
        try:
            response = await provider_adapter.invoke(request)
            
            # Add response attributes
            span.set_attribute("ai.model.tokens_in", response.usage.input_tokens)
            span.set_attribute("ai.model.tokens_out", response.usage.output_tokens)
            span.set_attribute("ai.model.finish_reason", response.stop_reason)
            span.set_attribute("ai.model.latency_ms", response.latency_ms)
            span.set_status(StatusCode.OK)
            
            return response
        except Exception as e:
            span.record_exception(e)
            span.set_status(StatusCode.ERROR, str(e))
            raise
```

---

## Pattern 3 — Agent Step Tracing

**Each LangGraph node must create a child span:**

```python
def create_traced_node(node_name: str, node_fn):
    tracer = trace.get_tracer("platform.agent-runtime")
    
    async def traced_node(state: AgentState) -> AgentState:
        parent_context = get_otel_context_from_state(state)
        
        with tracer.start_as_current_span(
            f"agent.step.{node_name}",
            context=parent_context
        ) as span:
            span.set_attribute("ai.agent.id", state["agent_id"])
            span.set_attribute("ai.agent.run_id", state["run_id"])
            span.set_attribute("ai.agent.step_number", state["step_count"])
            span.set_attribute("ai.agent.step_name", node_name)
            span.set_attribute("ai.tenant.id", state["tenant_id"])
            
            result = await node_fn(state)
            
            span.set_attribute("ai.agent.step_outcome", result.get("outcome", "success"))
            return result
    
    return traced_node
```

---

## Pattern 4 — SLO Metric Instruments

**Standard metric instruments for platform SLOs:**

```python
from opentelemetry import metrics

meter = metrics.get_meter("platform.slo-metrics")

# Counters
request_counter = meter.create_counter(
    "platform.requests.total",
    description="Total API requests",
    unit="1"
)

# Histograms (for latency)
latency_histogram = meter.create_histogram(
    "platform.request.latency",
    description="Request latency",
    unit="ms"
)

# Updown counters (for in-flight)
active_agents = meter.create_up_down_counter(
    "platform.agents.active",
    description="Currently running agent instances",
    unit="1"
)

# Observable gauges
token_budget_used = meter.create_observable_gauge(
    "platform.token_budget.used_pct",
    callbacks=[get_token_budget_usage],
    description="Token budget utilization percentage"
)

# Recording usage
def record_request(tenant_id, method, path, status, latency_ms):
    attrs = {"tenant.id": tenant_id, "http.method": method, "http.route": path, "http.status_code": status}
    request_counter.add(1, attrs)
    latency_histogram.record(latency_ms, attrs)
```

---

## Pattern 5 — Structured Logging

**All platform logs must be JSON structured:**

```python
import structlog

log = structlog.get_logger()

# Correct structured log
log.info(
    "agent.step.completed",
    agent_id=state["agent_id"],
    run_id=state["run_id"],
    step_name="document_analysis",
    step_number=state["step_count"],
    tenant_id=state["tenant_id"],
    latency_ms=latency_ms,
    tokens_consumed=tokens,
    outcome="success"
)

# Incorrect (never use)
print(f"Agent {agent_id} completed step with {tokens} tokens")
logger.info("Agent completed")  # No structure
```

**Never log:**
- API keys or credentials
- Full prompt content (hash only)
- PII (name, account numbers, etc.)
- Full response content (hash only)

---

## Pattern 6 — Alert Definitions

**Standard alert format for platform alerts:**

```yaml
# alerts/agent-runtime.yaml
groups:
  - name: agent-runtime
    rules:
      - alert: AgentRunFailureRateHigh
        expr: |
          rate(platform_agent_runs_total{outcome="error"}[5m]) /
          rate(platform_agent_runs_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
          team: ai-platform
        annotations:
          summary: "Agent run failure rate above 5%"
          description: "Tenant {{ $labels.tenant_id }} agent failure rate is {{ $value | humanizePercentage }}"
          runbook: "https://docs.platform/runbooks/agent-failure-rate"

      - alert: AgentRunFailureRateCritical
        expr: |
          rate(platform_agent_runs_total{outcome="error"}[5m]) /
          rate(platform_agent_runs_total[5m]) > 0.15
        for: 2m
        labels:
          severity: critical
          team: ai-platform
          pagerduty: "true"
        annotations:
          summary: "CRITICAL: Agent run failure rate above 15%"
```

---

## Observability Compliance Checklist

Before service goes to production:

- [ ] OTEL SDK initialized with correct resource attributes
- [ ] All service endpoints produce traces (auto or manual)
- [ ] AI-specific attributes on all AI operation spans
- [ ] Prometheus metrics exported (via OTEL or Prometheus client)
- [ ] Structured JSON logging configured
- [ ] No PII or secrets in logs, traces, or metrics
- [ ] Alerts defined for all SLO-relevant metrics
- [ ] Dashboard created in Grafana (before service ships)
- [ ] Runbook linked in alert annotations
