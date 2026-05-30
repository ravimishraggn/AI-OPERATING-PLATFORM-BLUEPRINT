# ADR-010 — Why HashiCorp Vault for Secrets Management

| Field | Value |
|---|---|
| **ADR Number** | 010 |
| **Title** | HashiCorp Vault as the Secrets and Key Management Platform |
| **Status** | Accepted |
| **Date** | 2026-05-30 |
| **Deciders** | Platform Architecture Team, Security Team |
| **Supersedes** | None |

---

## Context

The AI Operating Platform manages a large number of secrets:
- AI provider API keys (Anthropic, OpenAI, Azure OpenAI, Bedrock, Google)
- Database credentials (PostgreSQL, Redis, Neo4j)
- Kafka broker credentials
- Service-to-service mTLS certificates
- Tenant API keys
- Encryption keys for data at rest
- Third-party integration credentials

Security requirements:
- No secrets in source code or configuration files
- Dynamic secrets that rotate automatically (database passwords)
- Audit log of every secret access
- Multi-tenancy (tenant secrets isolated from platform secrets)
- PKI for mTLS certificate lifecycle
- Hardware Security Module (HSM) integration for regulated deployments
- Works on-premises and across clouds

---

## Decision

**We will use HashiCorp Vault (Business Source License) for secrets management and PKI.**

Deployment:
- **On-premises / Self-hosted:** Vault cluster on Kubernetes using official Helm chart
- **Cloud:** Vault on Kubernetes (same configuration), optionally using HCP Vault (managed)
- **Integration:** Vault Agent Injector sidecar pattern in Kubernetes

---

## Alternatives Considered

### Alternative 1 — AWS Secrets Manager
**Pros:**
- Fully managed
- Native AWS IAM integration
- Automatic rotation for AWS services

**Cons:**
- AWS-only; violates Cloud Agnostic principle (P4)
- Cannot run on-premises
- No PKI engine
- No dynamic secrets for non-AWS databases
- Significant per-secret cost at scale

**Verdict:** Rejected. Violates cloud-agnostic principle.

---

### Alternative 2 — Azure Key Vault
**Pros:**
- Fully managed
- Native Azure AD integration
- HSM-backed keys available

**Cons:**
- Azure-only; violates Cloud Agnostic principle (P4)
- Cannot run on-premises
- Limited multi-tenancy model
- Per-operation cost model

**Verdict:** Rejected. Violates cloud-agnostic principle.

---

### Alternative 3 — Kubernetes Secrets (native)
**Pros:**
- No additional service
- Natively supported by all Kubernetes tools

**Cons:**
- Secrets stored as base64 (not encrypted) by default
- etcd encryption required and complex to manage
- No dynamic secret rotation
- No audit log per-access
- No PKI engine
- No integration with HSM
- Not suitable for regulated industry secret management

**Verdict:** Rejected as primary. Kubernetes Secrets are used for non-sensitive configuration; sensitive secrets always come from Vault.

---

### Alternative 4 — CyberArk Conjur (OSS)
**Pros:**
- Open source (Conjur OSS)
- Designed for enterprise secrets
- Machine identity (similar to Vault AppRole)

**Cons:**
- Less ecosystem adoption than Vault
- Fewer Kubernetes integrations
- PKI engine less mature
- Smaller community

**Verdict:** Rejected. Vault's ecosystem advantage is significant.

---

### Alternative 5 — Infisical (OSS)
**Pros:**
- Open source (MIT)
- Modern developer experience
- Self-hostable

**Cons:**
- Less mature than Vault
- Dynamic secrets engine less comprehensive
- PKI engine not available
- Limited HSM support

**Verdict:** Rejected for production platform. Considered for developer experience tooling in Phase 6.

---

## Why Vault Wins

| Feature | HashiCorp Vault |
|---|---|
| Dynamic secrets | Yes (database, PKI, AWS, Azure, GCP) |
| Static secrets | Yes |
| PKI / Certificate engine | Yes (mTLS for service mesh) |
| Audit logging | Yes (every access logged) |
| Multi-tenancy | Vault namespaces |
| Kubernetes integration | Vault Agent Injector, CSI provider |
| HSM support | Yes (FIPS 140-2 Level 3) |
| License | BSL (Business Source License) |
| Self-hosted | Yes |
| On-premises | Yes |
| Cloud-agnostic | Yes |
| Transit secrets engine | Yes (encryption as a service) |

---

## Key Features Used

### 1 — Dynamic Database Credentials
Vault generates short-lived PostgreSQL credentials on request. No static database passwords exist.

```
Service → Vault Auth → Vault Database Engine → 
PostgreSQL Role Create → Credential (TTL: 1 hour) → Service
```

### 2 — AI Provider Key Management
AI provider API keys stored in Vault KV. Rotated via platform automation.

```
tenant_bankA/ai/anthropic_api_key
tenant_bankA/ai/openai_api_key
platform/ai/default_anthropic_key
```

### 3 — PKI Engine for mTLS
Vault PKI engine issues and renews certificates for all platform services.

```
Service → Vault PKI → Certificate (mTLS) → Service Identity
```

### 4 — Transit Engine (Encryption-as-a-Service)
Platform services encrypt sensitive data via Vault Transit. Keys never leave Vault.

```
Service → Vault Transit Encrypt(data) → Ciphertext → Store
Service → Store → Vault Transit Decrypt(ciphertext) → Plaintext
```

### 5 — Vault Namespaces for Multi-tenancy
```
vault/
├── namespace: platform/    ← Platform infrastructure secrets
├── namespace: tenant-bankA/ ← Bank A tenant secrets
├── namespace: tenant-bankB/ ← Bank B tenant secrets
└── namespace: tenant-insurer/ ← Insurer tenant secrets
```

---

## Vault Agent Injector Pattern

In Kubernetes, the Vault Agent Injector is used to automatically inject secrets into pods without application code changes:

```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "loan-service"
  vault.hashicorp.com/agent-inject-secret-db: "database/creds/loan-service-role"
```

The pod gets a sidecar that authenticates to Vault and writes secrets to a shared volume. Application reads secrets from files, not environment variables (more secure).

---

## License Note

HashiCorp Vault moved from MPL 2.0 to Business Source License (BSL) in August 2023. The BSL allows free use for all non-competing production workloads. OpenBao is the community-maintained MPL fork of Vault. The platform architecture supports switching to OpenBao if licensing becomes a concern.

---

## Consequences

### Positive
- Dynamic secrets eliminate long-lived credentials (major security improvement)
- Every secret access audited — regulatory compliance friendly
- PKI engine manages mTLS certificate lifecycle automatically
- Vault namespaces provide clean multi-tenant secret isolation
- Transit engine provides encryption-as-a-service without key management burden

### Negative
- Vault adds operational complexity (HA cluster, unsealing, backup)
- BSL license (though production use for non-competing products is free)
- Vault must be highly available — if Vault is down, services cannot start

---

## Tradeoffs

| Gain | Cost |
|---|---|
| Dynamic, short-lived credentials | Vault HA cluster operational overhead |
| Complete secret access audit trail | BSL license (not fully OSS) |
| PKI engine eliminates manual cert management | Must be highly available (critical path) |
| Multi-tenant secret isolation | Learning curve for teams new to Vault |
