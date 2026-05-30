# Skill: Governance Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Governance Plane, Security Plane, Trust Plane
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable patterns for implementing AI governance controls in platform components. These patterns ensure consistent governance across all AI operations.

---

## Pattern 1 — Policy Guard (Synchronous Policy Check)

**When to use:** Before any operation that must be validated against business or compliance rules. Blocks operation if policy violated.

```python
class PolicyGuardMiddleware:
    async def __call__(self, operation: Operation, context: PlatformContext) -> Operation:
        # Evaluate against all active policies
        result = await opa_client.evaluate(
            package="platform.operations",
            input={
                "operation": operation.type,
                "actor": context.actor,
                "resource": operation.resource,
                "tenant_id": context.tenant_id,
                "data_classification": operation.data_classification,
                "timestamp": datetime.utcnow().isoformat()
            }
        )
        
        if not result.allow:
            # Log policy denial (immutable)
            await audit_service.record_denial(
                operation=operation,
                violated_policies=result.violated_policies,
                context=context
            )
            raise PolicyViolationError(
                f"Operation denied: {result.violated_policies}"
            )
        
        # Log policy approval
        await audit_service.record_approval(operation, context)
        return operation
```

---

## Pattern 2 — Audit Decorator (Every AI Operation)

**When to use:** Wrap every AI-significant operation for automatic audit.

```python
def audit_operation(event_type: str):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            context = extract_platform_context(args, kwargs)
            start_time = time.monotonic()
            
            try:
                result = await func(*args, **kwargs)
                await kafka_producer.send(
                    topic=f"platform.{context.tenant_id}.governance.audit",
                    value=AuditEvent(
                        event_type=event_type,
                        actor=context.actor,
                        tenant_id=context.tenant_id,
                        outcome="success",
                        latency_ms=(time.monotonic() - start_time) * 1000,
                        metadata=extract_audit_metadata(result)
                    ).model_dump_json()
                )
                return result
            except Exception as e:
                await kafka_producer.send(
                    topic=f"platform.{context.tenant_id}.governance.audit",
                    value=AuditEvent(
                        event_type=event_type,
                        outcome="error",
                        error=str(e),
                        ...
                    ).model_dump_json()
                )
                raise
        return wrapper
    return decorator

# Usage:
@audit_operation("agent.step.completed")
async def execute_agent_step(state: AgentState) -> AgentState:
    ...
```

---

## Pattern 3 — Lineage Chain (Data Provenance)

**When to use:** Any time data moves from one system to another or is transformed by AI.

```python
class LineageTracker:
    async def record_transformation(
        self,
        input_entity: str,         # "doc:uuid-001"
        output_entity: str,        # "chunk:uuid-001-c3"
        transformation: str,       # "document_chunking"
        parameters: dict,          # {"strategy": "semantic", "chunk_size": 512}
        tenant_id: str
    ) -> str:
        """Returns lineage_id"""
        lineage_event = OpenLineageEvent(
            eventType="COMPLETE",
            inputs=[DatasetFacets(namespace="platform", name=input_entity)],
            outputs=[DatasetFacets(namespace="platform", name=output_entity)],
            job=Job(namespace="platform", name=transformation),
            run=Run(facets={"parameters": parameters}),
            tenant_id=tenant_id
        )
        return await marquez_client.record(lineage_event)
```

---

## Pattern 4 — Consent Gate

**When to use:** Consumer data used in AI decisions where consent may be required.

```python
async def check_consent_gate(
    subject_id: str,
    processing_purpose: str,
    tenant_id: str
) -> ConsentDecision:
    consent = await consent_service.get_consent(
        subject_id=subject_id,
        purpose=processing_purpose,
        tenant_id=tenant_id
    )
    
    if not consent.granted:
        if consent.required:
            raise ConsentRequiredError(
                f"Processing requires consent for purpose: {processing_purpose}"
            )
        else:
            # Log processing without consent (may be legitimate interest basis)
            await audit_service.log_processing_without_consent(
                subject_id=subject_id,
                purpose=processing_purpose,
                legal_basis=consent.legal_basis
            )
    
    return consent
```

---

## Pattern 5 — Immutable Decision Record

**When to use:** Any AI-influenced decision that could be contested, audited, or regulated.

```python
@dataclass(frozen=True)  # Immutable after creation
class DecisionRecord:
    decision_id: str
    timestamp: datetime
    decision_type: str
    tenant_id: str
    subject_id: str
    outcome: str
    confidence: float
    
    # What data was used?
    input_data_refs: tuple[str, ...]  # lineage IDs, not actual data
    
    # What model made the decision?
    model_id: str
    model_version: str
    
    # What was the reasoning?
    reasoning_steps: tuple[dict, ...]
    
    # Human review outcome (if applicable)
    human_review: dict | None
    
    # Can it be contested?
    contestable: bool = True
    contest_deadline: datetime | None = None
    
    def to_signed_record(self, signing_key: bytes) -> str:
        """Produces HMAC-signed JSON for tamper-evident storage"""
        record_json = json.dumps(asdict(self), default=str, sort_keys=True)
        signature = hmac.new(signing_key, record_json.encode(), hashlib.sha256).hexdigest()
        return json.dumps({"record": record_json, "signature": signature})
```

---

## Pattern 6 — Erasure Propagation

**When to use:** GDPR right-to-erasure request received.

```python
class ErasureOrchestrator:
    async def execute_erasure(self, subject_id: str, tenant_id: str) -> ErasureReport:
        tasks = [
            self.erase_from_postgresql(subject_id, tenant_id),
            self.erase_from_qdrant(subject_id, tenant_id),
            self.erase_from_neo4j(subject_id, tenant_id),
            self.revoke_kafka_encryption_key(subject_id, tenant_id),
            self.archive_audit_records(subject_id, tenant_id),  # Retain for legal
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Record erasure completion (audit record preserved)
        await audit_service.record_erasure_completion(
            subject_id=subject_id,
            tenant_id=tenant_id,
            results=results
        )
        
        return ErasureReport(
            subject_id=subject_id,
            completed_at=datetime.utcnow(),
            systems_erased=len([r for r in results if not isinstance(r, Exception)]),
            errors=[str(r) for r in results if isinstance(r, Exception)]
        )
```

---

## Governance Checklist (Per Feature)

Before any AI feature ships:

| Check | Owner | Verification |
|---|---|---|
| Policy defined for this operation type | Governance Team | OPA policy exists and tested |
| Audit event defined and published | Engineering | Kafka topic, schema |
| Lineage tracked end-to-end | Engineering | OpenLineage events emitted |
| Explainability available | Engineering | /explanation endpoint responds |
| Data classification applied | Data Team | All input data classified |
| Consent gate (if consumer data) | Legal | ConsentGate middleware in place |
| HITL gate (if high-stakes) | Product | HITLConditions configured |
| Erasure support | Engineering | ErasureOrchestrator handles data |
| Retention policy set | Compliance | Retention rules in policy store |
