# Core Platform Ontology

> **Document Type:** Ontology Definition
> **Status:** Draft
> **Owner:** Knowledge Engineering Team
> **Last Updated:** 2026-05-30

---

## Purpose

The Core Platform Ontology defines the fundamental concepts of the AI Operating Platform itself. Every domain ontology (banking, insurance, healthcare) extends this core ontology. These concepts are consistent across all tenants and all deployments.

---

## Core Concepts (Classes)

### Platform Actor

```
PlatformActor (abstract)
  ├── HumanUser
  │     ├── PlatformAdmin
  │     ├── TenantAdmin
  │     └── AIMengineer
  ├── AIAgent
  │     ├── ReactiveAgent
  │     ├── DeliberativeAgent
  │     └── AutonomousAgent
  └── PlatformService
```

### AI Operation

```
AIOperation (abstract)
  ├── ModelInvocation
  ├── EmbeddingGeneration
  ├── VectorSearch
  ├── GraphQuery
  ├── AgentRun
  │     └── AgentStep
  └── WorkflowExecution
```

### Artifact

```
Artifact (abstract)
  ├── AIModel
  │     ├── LanguageModel
  │     ├── EmbeddingModel
  │     └── ClassificationModel
  ├── Agent (definition)
  ├── PromptTemplate
  ├── MCPTool
  ├── Skill
  ├── EvaluationDataset
  └── WorkflowDefinition
```

### Knowledge Entity

```
KnowledgeEntity (abstract)
  ├── Document
  ├── DocumentChunk
  ├── Organization
  ├── Person
  ├── Product
  ├── Regulation
  ├── Policy
  └── Event
```

### Decision

```
Decision (abstract)
  ├── AIDecision
  │     ├── RecommendationDecision
  │     ├── ClassificationDecision
  │     └── ApprovalDecision
  └── HumanDecision
        ├── HITLDecision
        └── OverrideDecision
```

### Governance Record

```
GovernanceRecord (abstract)
  ├── AuditEvent
  ├── PolicyEvaluationResult
  ├── LineageRecord
  ├── ComplianceReport
  └── DecisionRecord
```

---

## Core Properties

| Property | Domain | Range | Description |
|---|---|---|---|
| `platform:id` | All | xsd:string | Platform-assigned unique identifier |
| `platform:tenantId` | All | xsd:string | Owning tenant identifier |
| `platform:createdAt` | All | xsd:dateTime | Creation timestamp |
| `platform:createdBy` | All | PlatformActor | Creating actor |
| `platform:status` | Artifact | xsd:string | Lifecycle status |
| `platform:version` | Artifact | xsd:string | Semantic version |
| `platform:tags` | All | xsd:string | Descriptive tags |

---

## Core Relationships

```
AIAgent --[uses]--> AIModel
AIAgent --[has_capability]--> MCPTool
AIAgent --[produces]--> AIDecision
AIAgent --[generates]--> AuditEvent
AIAgent --[accesses]--> KnowledgeEntity

AgentRun --[executes]--> AIAgent
AgentRun --[has_step]--> AgentStep
AgentRun --[may_require]--> HITLDecision
AgentRun --[checkpoints_to]--> GovernanceRecord

ModelInvocation --[uses]--> AIModel
ModelInvocation --[generates]--> AuditEvent
ModelInvocation --[has_cost]--> xsd:decimal

KnowledgeEntity --[has_provenance]--> LineageRecord
DocumentChunk --[is_part_of]--> Document
DocumentChunk --[has_embedding]--> VectorEmbedding
```

---

## Ontology File Locations

```
docs/ontologies/
├── core-ontology.md           ← This document (conceptual)
├── core-ontology.ttl          ← Turtle/RDF formal representation (Phase 3)
├── domain-ontologies.md       ← Domain extension guidance
└── domain/
    ├── banking-ontology.md    ← FIBO-based banking concepts
    ├── insurance-ontology.md  ← ACORD-based insurance concepts
    └── healthcare-ontology.md ← FHIR/SNOMED healthcare concepts
```

---

## Ontology Governance

- Core ontology changes require Architecture Team approval
- Domain ontology changes require Tenant Admin approval + Architecture review
- All ontology changes versioned (breaking changes = major version bump)
- Formal OWL/Turtle files generated from conceptual definitions in Phase 3
