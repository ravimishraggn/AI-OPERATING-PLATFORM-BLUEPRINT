# Skill: Platform Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Platform-wide
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable architectural patterns that apply across all platform services. These are the cross-cutting patterns that ensure consistency, reliability, and correctness across the entire platform.

---

## Pattern 1 — Provider Adapter (AI Model Abstraction)

**All AI model calls go through an adapter implementing this interface:**

```python
from abc import ABC, abstractmethod
from typing import AsyncIterator

class ModelAdapter(ABC):
    """Abstract base for all AI provider adapters."""
    
    @abstractmethod
    async def invoke(self, request: ModelRequest) -> ModelResponse:
        """Non-streaming model invocation."""
        pass
    
    @abstractmethod
    async def stream(self, request: ModelRequest) -> AsyncIterator[ModelChunk]:
        """Streaming model invocation."""
        pass
    
    @abstractmethod
    async def health_check(self) -> AdapterHealth:
        """Check provider availability."""
        pass
    
    @property
    @abstractmethod
    def provider_id(self) -> str:
        """e.g., 'anthropic', 'openai', 'bedrock'"""
        pass

# Concrete adapter
class AnthropicAdapter(ModelAdapter):
    async def invoke(self, request: ModelRequest) -> ModelResponse:
        api_key = await vault_client.get_secret(f"ai/anthropic/{request.tenant_id}")
        client = anthropic.AsyncAnthropic(api_key=api_key)
        
        message = await client.messages.create(
            model=request.model_id,
            max_tokens=request.max_tokens,
            messages=request.messages,
            system=request.system_prompt
        )
        
        return ModelResponse(
            content=message.content[0].text,
            usage=TokenUsage(
                input_tokens=message.usage.input_tokens,
                output_tokens=message.usage.output_tokens
            ),
            stop_reason=message.stop_reason,
            model=message.model
        )
    
    @property
    def provider_id(self) -> str:
        return "anthropic"
```

---

## Pattern 2 — Circuit Breaker (Provider Resilience)

**Prevent cascade failures when a provider is unavailable:**

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self.last_failure_time = None
    
    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Circuit breaker OPEN")
        
        try:
            result = await func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                log.warning("Circuit breaker OPEN", provider=self.provider_id)
            raise

# Usage in Model Router
async def invoke_with_fallback(request: ModelRequest) -> ModelResponse:
    for provider in request.fallback_chain:
        try:
            return await circuit_breakers[provider].call(
                adapters[provider].invoke, request
            )
        except (CircuitBreakerOpenError, ProviderError) as e:
            log.warning("Provider failed, trying next", provider=provider, error=str(e))
    
    raise AllProvidersFailedError("All providers in fallback chain failed")
```

---

## Pattern 3 — Outbox Pattern (Reliable Event Publishing)

**Guarantee event delivery without 2-phase commit:**

```python
# Instead of: save to DB AND publish to Kafka (can fail between the two)
# Use: save to DB with outbox entry, separate outbox publisher

async def save_decision_with_outbox(decision: Decision, session: AsyncSession):
    # Single database transaction
    async with session.begin():
        # Save the business entity
        session.add(DecisionEntity.from_domain(decision))
        
        # Save outbox entry (same transaction)
        session.add(OutboxEvent(
            event_type="decision.created",
            payload=decision.model_dump_json(),
            topic=f"platform.{decision.tenant_id}.decisions",
            created_at=datetime.utcnow()
        ))
        # If this transaction commits, the outbox entry exists
        # Outbox publisher will pick it up and publish to Kafka

# Outbox publisher (runs as background service)
async def outbox_publisher():
    while True:
        events = await session.execute(
            select(OutboxEvent)
            .where(OutboxEvent.published_at == None)
            .order_by(OutboxEvent.created_at)
            .limit(100)
            .with_for_update(skip_locked=True)
        )
        
        for event in events:
            await kafka_producer.send(event.topic, event.payload)
            event.published_at = datetime.utcnow()
        
        await session.commit()
        await asyncio.sleep(1)
```

---

## Pattern 4 — Idempotency Key

**Prevent duplicate operations (especially for AI agent runs and billing):**

```python
class IdempotencyLayer:
    def __init__(self, redis: Redis, ttl_seconds: int = 86400):
        self.redis = redis
        self.ttl = ttl_seconds
    
    async def execute_once(
        self,
        idempotency_key: str,
        operation: Callable,
        *args, **kwargs
    ) -> Any:
        # Check if already executed
        cached = await self.redis.get(f"idempotency:{idempotency_key}")
        if cached:
            return json.loads(cached)
        
        # Execute and cache result
        result = await operation(*args, **kwargs)
        
        await self.redis.setex(
            f"idempotency:{idempotency_key}",
            self.ttl,
            json.dumps(result, default=str)
        )
        
        return result

# Usage: Agent run endpoint
@router.post("/agents/{agent_id}/runs")
async def run_agent(
    agent_id: str,
    request: AgentRunRequest,
    x_idempotency_key: str = Header(None),
    ctx: PlatformContext = Depends(get_platform_context)
):
    if x_idempotency_key:
        return await idempotency.execute_once(
            idempotency_key=f"{ctx.tenant_id}:{x_idempotency_key}",
            operation=agent_runner.run,
            agent_id=agent_id, request=request, ctx=ctx
        )
    return await agent_runner.run(agent_id=agent_id, request=request, ctx=ctx)
```

---

## Pattern 5 — Tenant Context Propagation

**Pass tenant context through the entire call chain:**

```python
# Python: context var approach
from contextvars import ContextVar

_tenant_context: ContextVar[TenantContext] = ContextVar("tenant_context")

def get_tenant_context() -> TenantContext:
    return _tenant_context.get()

def set_tenant_context(ctx: TenantContext) -> Token:
    return _tenant_context.set(ctx)

# FastAPI dependency
async def tenant_context_middleware(request: Request, call_next):
    jwt_context = extract_jwt_context(request)
    token = set_tenant_context(TenantContext(
        tenant_id=jwt_context.tenant_id,
        actor_id=jwt_context.sub,
        roles=jwt_context.roles
    ))
    try:
        response = await call_next(request)
        return response
    finally:
        _tenant_context.reset(token)

# All downstream code reads from context (no parameter passing needed)
async def query_knowledge_graph(query: str) -> list[Entity]:
    ctx = get_tenant_context()  # Available anywhere in the call chain
    return await neo4j.query(query, tenant_id=ctx.tenant_id)
```

---

## Pattern 6 — Health Check Standard

**Every service exposes a structured health check:**

```python
@router.get("/health")
async def health_check() -> HealthStatus:
    checks = await asyncio.gather(
        check_database(),
        check_kafka(),
        check_vault(),
        check_redis(),
        return_exceptions=True
    )
    
    results = {
        "database": checks[0],
        "kafka": checks[1],
        "vault": checks[2],
        "redis": checks[3]
    }
    
    overall = "healthy" if all(c == "healthy" for c in results.values()) else "degraded"
    
    return HealthStatus(
        status=overall,
        version=os.getenv("SERVICE_VERSION"),
        checks=results,
        timestamp=datetime.utcnow()
    )

@router.get("/ready")  # Kubernetes readiness probe
async def readiness() -> dict:
    # Ready only if critical dependencies are available
    if not await vault_client.is_available():
        raise HTTPException(503, "Vault unavailable")
    if not await db.is_available():
        raise HTTPException(503, "Database unavailable")
    return {"ready": True}

@router.get("/live")  # Kubernetes liveness probe
async def liveness() -> dict:
    return {"alive": True}  # Simple: if service can respond, it's alive
```

---

## API Versioning Standard

```
All platform APIs use URL path versioning:
  /api/v1/agents/{id}     ← Current version
  /api/v2/agents/{id}     ← New version (breaking change)

Deprecation lifecycle:
  1. New version released alongside old version
  2. Old version marked deprecated in OpenAPI spec
  3. Deprecation header added: "Deprecation: true; Sunset: 2027-01-01"
  4. 6 months notice before removal
  5. Old version removed
```

---

## Error Response Standard

```python
# All errors return this structure
class PlatformError(BaseModel):
    error_code: str       # e.g., "AGENT_NOT_FOUND", "POLICY_VIOLATION"
    message: str          # Human-readable message
    details: dict = {}    # Additional context
    trace_id: str         # OpenTelemetry trace ID for debugging
    timestamp: datetime

# Error codes follow pattern: {DOMAIN}_{CONDITION}
# Examples:
#   AGENT_NOT_FOUND, AGENT_QUOTA_EXCEEDED
#   MODEL_PROVIDER_UNAVAILABLE, MODEL_RATE_LIMIT_EXCEEDED
#   GOVERNANCE_POLICY_VIOLATION, GOVERNANCE_CONSENT_REQUIRED
#   SECURITY_UNAUTHORIZED, SECURITY_FORBIDDEN
```
