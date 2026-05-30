# ADR-002 — Why C# for Platform Services

| Field | Value |
|---|---|
| **ADR Number** | 002 |
| **Title** | C# / ASP.NET Core for Platform Services Layer |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team |
| **Supersedes** | None |

---

## Context

The platform services layer — comprising REST APIs, gRPC services, background workers, multi-tenant management, authentication, authorization, and workflow coordination — requires a language and runtime choice that will govern the platform for years.

This layer handles:
- Platform control APIs (tenant management, agent registration, policy enforcement)
- Authentication and authorization middleware
- Workflow orchestration coordination
- Event streaming consumers/producers (Kafka)
- Background processing (evaluation jobs, audit aggregation)
- gRPC services for high-throughput inter-service communication

The AI runtime layer (Python/FastAPI) is decided separately (ADR-003). This decision is about the platform services that surround and govern the AI runtime.

---

## Decision

**We will use C# with ASP.NET Core for all platform services.**

---

## Alternatives Considered

### Alternative 1 — Python (FastAPI or Django)
**Pros:**
- Same language as AI runtime (fewer context switches)
- Large developer pool familiar with Python
- FastAPI is excellent for async APIs

**Cons:**
- Python's type system is optional and inconsistent in practice
- GIL limits true multi-threaded CPU-bound work
- Memory usage per request higher than .NET
- Enterprise middleware ecosystem (auth, multi-tenancy, gRPC, Kafka) less mature than .NET
- Python is chosen for the AI runtime; mixing in platform services creates a monolingual risk

**Verdict:** Rejected for platform services. Python is chosen for AI runtime (ADR-003); platform services need different characteristics.

---

### Alternative 2 — Java / Spring Boot
**Pros:**
- Extremely mature enterprise ecosystem
- Excellent Kafka, gRPC, security libraries
- Strong type system

**Cons:**
- Higher memory footprint than .NET
- Slower startup time (significant for Kubernetes sidecar patterns)
- Historically verbose; productivity lower than C#
- Target organization does not have strong Java expertise
- GraalVM native compilation reduces portability

**Verdict:** Rejected. Technical merits are strong but organizational fit is lower than C#.

---

### Alternative 3 — Go
**Pros:**
- Excellent performance and low memory usage
- Statically compiled, fast startup
- Good Kubernetes and cloud-native tooling (many k8s operators written in Go)

**Cons:**
- Error handling verbosity adds maintenance burden
- Generics are recent and ecosystem adoption is uneven
- Enterprise patterns (DI, multi-tenancy, auth) require more manual implementation
- Limited organizational expertise

**Verdict:** Rejected. Strong for infrastructure tooling, insufficient for enterprise service development.

---

### Alternative 4 — Node.js / TypeScript
**Pros:**
- TypeScript provides good type safety
- Shared language with frontend (Next.js)
- Large ecosystem

**Cons:**
- Single-threaded event loop not ideal for CPU-intensive orchestration
- Enterprise patterns less standardized than .NET
- Memory management less predictable
- gRPC support less mature than .NET

**Verdict:** Rejected for platform services. TypeScript is chosen for frontend.

---

## Why C# Wins

| Criterion | C# / .NET | Python | Java | Go |
|---|---|---|---|---|
| Type Safety | Strong | Weak | Strong | Strong |
| Enterprise Patterns | Excellent | Good | Excellent | Moderate |
| gRPC Support | First-class | Good | Good | First-class |
| Kafka Integration | Excellent | Good | Excellent | Good |
| Performance | Excellent | Moderate | Good | Excellent |
| Memory Efficiency | Excellent | Moderate | Moderate | Excellent |
| Async/Await | Native | Native | Reactor | Goroutines |
| Startup Time (AOT) | Excellent | Moderate | Slow | Excellent |
| DI Container | Built-in | 3rd party | Spring | Manual |
| Multi-tenancy Patterns | Rich | Adequate | Rich | Manual |
| Organizational Fit | High | Medium | Low | Low |

---

## Consequences

### Positive
- ASP.NET Core's built-in DI, middleware pipeline, and health checks reduce boilerplate
- .NET 8+ AOT compilation enables fast Kubernetes pod startup
- Excellent Confluent Kafka .NET client
- MassTransit for saga orchestration patterns
- Strong gRPC / Protobuf support via grpc-dotnet
- Native OpenTelemetry instrumentation
- C# type system enables contract-first API development

### Negative
- Two languages in the platform (C# services + Python AI runtime) require polyglot team skills
- Container images slightly larger than Go equivalents
- Developers must manage both ecosystems

### Mitigations
- Clear boundary: C# for platform control plane; Python for AI execution plane
- Shared OpenAPI specs and Protobuf definitions as language-neutral contracts
- Consistent OpenTelemetry setup across both runtimes

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Enterprise-grade middleware | Polyglot team requirement |
| Strong typing and performance | Two language ecosystems to maintain |
| Rich .NET ecosystem | C# expertise required alongside Python |
| Native async, gRPC, Kafka support | Learning curve for Python-native teams |
