# Naming Conventions — Enterprise AI Operating Platform

> **Document Type:** Platform Standards
> **Status:** Accepted
> **Owner:** Platform Architecture Team
> **Last Updated:** 2026-05-30

---

## Purpose

Consistent naming across the platform reduces cognitive load, prevents conflicts, and enables automated tooling. These conventions are mandatory for all platform components.

---

## Kubernetes Resources

```
Namespaces:      {scope}-{name}
                 platform-system, tenant-bankA, observability

Services:        {plane}-{function}-svc
                 model-router-svc, agent-runtime-svc

Deployments:     {plane}-{function}
                 model-router, agent-runtime, governance-api

ConfigMaps:      {service}-config
                 model-router-config, agent-runtime-config

Secrets:         {service}-secret (Vault-injected; no literal secrets)

ServiceAccounts: {service}-sa
                 model-router-sa, agent-runtime-sa

Labels (required):
  app.kubernetes.io/name: {service}
  app.kubernetes.io/part-of: ai-operating-platform
  app.kubernetes.io/version: {semver}
  platform.ai/plane: {plane-number}
  platform.ai/tenant: {tenant-id}
```

---

## Kafka Topics

```
Pattern: platform.{tenant_id}.{domain}.{event_type}

Examples:
  platform.tenant-bankA.agent.events
  platform.tenant-bankA.agent.checkpoints
  platform.tenant-bankA.governance.audit
  platform.tenant-bankA.data.ingestion
  platform.tenant-bankA.evaluation.results
  platform.system.alerts
  platform.system.metrics
```

---

## Database Objects

```
PostgreSQL schemas:   tenant_{tenant_id}
                      tenant_banka, tenant_bankb

Tables:               {domain}_{entity_plural}
                      agent_runs, governance_audit_events, model_invocations

Indexes:              idx_{table}_{columns}
                      idx_agent_runs_tenant_id_created_at

Primary keys:         id (UUID)

Foreign keys:         {referenced_table}_id
```

---

## API Endpoints

```
Pattern: /api/v{version}/{plane-noun}/{resource}/{identifier}/{sub-resource}

Examples:
  /api/v1/agents
  /api/v1/agents/{agent_id}
  /api/v1/agents/{agent_id}/runs
  /api/v1/agents/runs/{run_id}
  /api/v1/models/invoke
  /api/v1/governance/audit/events
```

---

## Vault Paths

```
Platform secrets:   platform/{domain}/{key}
                    platform/ai/anthropic_api_key_default

Tenant secrets:     tenant-{id}/{domain}/{key}
                    tenant-bankA/ai/anthropic_api_key
                    tenant-bankA/db/connection_string

Dynamic secrets:    database/creds/{role}
                    database/creds/tenant-bankA-read-role
```

---

## Code Conventions

### Python
```
modules:        snake_case (agent_runtime.py)
classes:        PascalCase (AgentExecutor)
functions:      snake_case (execute_agent_step)
constants:      UPPER_SNAKE_CASE (MAX_RETRY_COUNT)
type aliases:   PascalCase (TenantId = str)
```

### C#
```
namespaces:     PascalCase (Platform.ModelPlane)
classes:        PascalCase (ModelRouter)
methods:        PascalCase (InvokeModel)
private fields: _camelCase (_vaultClient)
constants:      PascalCase (MaxTokenBudget)
interfaces:     IPascalCase (IModelAdapter)
```

---

## Artifact IDs

```
Agents:          {domain}-{function}-agent[-v{major}]
                 loan-underwriting-agent-v2, document-analysis-agent

Prompts:         {domain}-{function}-{type}
                 loan-underwriting-system, risk-analysis-user

MCP Tools:       {domain}-{function}-tool
                 knowledge-graph-query-tool, vector-search-tool

Workflows:       {domain}-{process}-workflow[-v{major}]
                 loan-application-underwriting-workflow-v3

Models (platform ID): {provider}-{model-family}-{size}
                 anthropic-claude-opus-large, openai-gpt4o-standard
```

---

## Environment Names

```
local     - Developer local machine (Docker Compose)
dev       - Shared development cluster
staging   - Pre-production validation cluster
prod      - Production cluster
dr        - Disaster recovery cluster (standby)
```

---

## OTEL Attribute Naming

```
All custom attributes prefixed:  ai.* or platform.*

Required:
  ai.model.provider       (anthropic, openai, bedrock, gemini, ollama)
  ai.model.id             (claude-opus-4-8, gpt-4o, etc.)
  ai.tenant.id            (tenant-bankA)
  ai.agent.id             (loan-underwriting-agent-v2)
  ai.agent.run_id         (run-uuid)
  ai.agent.step_name      (document_analysis)
  platform.plane          (06-agent-runtime)
  platform.environment    (prod, staging)
```
