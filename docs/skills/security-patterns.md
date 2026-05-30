# Skill: Security Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Security Plane
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable security patterns for all platform services. These patterns implement the Security Principles and must be applied consistently.

---

## Pattern 1 — JWT Validation Middleware

**Every service validates the platform JWT:**

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer
import jwt

security = HTTPBearer()

class PlatformContext:
    def __init__(self, sub: str, tenant_id: str, roles: list[str], permissions: list[str]):
        self.sub = sub
        self.tenant_id = tenant_id
        self.roles = roles
        self.permissions = permissions

async def get_platform_context(token: HTTPAuthorizationCredentials = Security(security)) -> PlatformContext:
    try:
        payload = jwt.decode(
            token.credentials,
            key=await vault_client.get_jwt_public_key(),  # From Vault
            algorithms=["RS256"],
            options={"require": ["sub", "tenant_id", "exp", "iat"]}
        )
        return PlatformContext(
            sub=payload["sub"],
            tenant_id=payload["tenant_id"],
            roles=payload.get("roles", []),
            permissions=payload.get("permissions", [])
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")

# Usage on every endpoint
@router.post("/agents/{agent_id}/runs")
async def run_agent(
    agent_id: str,
    request: AgentRunRequest,
    ctx: PlatformContext = Depends(get_platform_context)
):
    # ctx.tenant_id is verified from JWT, cannot be spoofed
    ...
```

---

## Pattern 2 — Tenant Scope Enforcement

**Prevent cross-tenant data access:**

```python
class TenantScopedRepository:
    def __init__(self, db: AsyncSession, tenant_id: str):
        self._db = db
        self._tenant_id = tenant_id
    
    async def get_agent(self, agent_id: str) -> Agent:
        # ALWAYS filter by tenant_id — never trust agent_id alone
        agent = await self._db.execute(
            select(Agent)
            .where(Agent.id == agent_id)
            .where(Agent.tenant_id == self._tenant_id)  # Critical
        )
        if not agent:
            raise NotFoundError(f"Agent {agent_id} not found")  # Don't leak cross-tenant existence
        return agent
```

**C# pattern:**
```csharp
// TenantScopedDbContext adds tenant filter to every query
public class TenantScopedDbContext : DbContext
{
    private readonly string _tenantId;
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Agent>()
            .HasQueryFilter(a => a.TenantId == _tenantId);
        // HasQueryFilter ensures tenant_id is added to every SELECT
    }
}
```

---

## Pattern 3 — Input Sanitization for AI

**Before any user input reaches an AI model:**

```python
class InputSanitizer:
    INJECTION_PATTERNS = [
        r"ignore\s+(all\s+)?previous\s+instructions",
        r"you\s+are\s+now\s+(an?\s+)?(?:unrestricted|jailbroken|evil)",
        r"pretend\s+you\s+(are|have\s+no)",
        r"</?(system|assistant|human)>",  # XML injection
        r"\[INST\].*\[/INST\]",  # Llama template injection
    ]
    
    async def sanitize(self, text: str) -> SanitizationResult:
        issues = []
        
        # Pattern-based injection detection
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, text, re.IGNORECASE):
                issues.append(f"Potential injection: {pattern}")
        
        # ML-based injection scoring
        score = await self.injection_classifier.score(text)
        if score > 0.7:
            issues.append(f"High injection probability: {score:.2f}")
        
        # PII detection
        pii_entities = await presidio.analyze(text, language="en")
        
        return SanitizationResult(
            sanitized_text=text,  # Don't modify; let caller decide
            has_issues=len(issues) > 0,
            issues=issues,
            injection_score=score,
            pii_entities=pii_entities
        )
```

---

## Pattern 4 — Secret Retrieval (Never in Code)

```python
class VaultSecretProvider:
    """All secrets come from Vault. Never from environment variables or config files."""
    
    async def get_ai_api_key(self, provider: str, tenant_id: str) -> str:
        # Vault path: tenant-{id}/ai/{provider}_api_key
        secret = await vault_client.read(
            path=f"tenant-{tenant_id}/ai/{provider}_api_key",
            cache_ttl=60  # Cache for 60 seconds
        )
        return secret.data["value"]
    
    async def get_db_credentials(self, service: str, tenant_id: str) -> DBCredentials:
        # Dynamic credentials from Vault database engine
        creds = await vault_client.generate_dynamic_secret(
            path=f"database/creds/{service}-{tenant_id}-role"
        )
        return DBCredentials(
            username=creds.data["username"],
            password=creds.data["password"],
            lease_duration=creds.lease_duration
        )
```

---

## Pattern 5 — Output Sanitization

**After any AI model response:**

```python
class OutputSanitizer:
    SENSITIVE_PATTERNS = [
        r"(?i)(api[_\-\s]?key|secret|password|token)\s*[:=]\s*\S+",  # Credential exposure
        r"\b\d{16}\b",  # Credit card numbers
        r"(?i)(ssn|social\s+security)\s*:?\s*\d{3}-\d{2}-\d{4}",  # SSN
        r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",  # Email (if not expected)
    ]
    
    async def sanitize_output(self, text: str, allowed_pii_types: list[str] = []) -> str:
        # Run Presidio on output
        pii_results = await presidio.analyze(text, language="en")
        
        # Anonymize any PII not in the allowed list
        for entity in pii_results:
            if entity.entity_type not in allowed_pii_types:
                text = text[:entity.start] + f"[{entity.entity_type}]" + text[entity.end:]
        
        # Check for accidental credential exposure
        for pattern in self.SENSITIVE_PATTERNS:
            if re.search(pattern, text):
                log.security_alert("Potential credential in AI output", pattern=pattern)
                text = re.sub(pattern, "[REDACTED]", text)
        
        return text
```

---

## Pattern 6 — Rate Limiting (Redis Token Bucket)

```python
class RateLimiter:
    def __init__(self, redis: Redis):
        self.redis = redis
    
    async def check_rate_limit(
        self,
        tenant_id: str,
        resource: str,
        limit: int,
        window_seconds: int
    ) -> RateLimitResult:
        key = f"rate_limit:{tenant_id}:{resource}"
        
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - window_seconds
        
        # Sliding window algorithm
        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, window_seconds + 1)
        
        _, _, count, _ = await pipe.execute()
        
        remaining = max(0, limit - count)
        allowed = count <= limit
        
        return RateLimitResult(
            allowed=allowed,
            limit=limit,
            remaining=remaining,
            reset_after=window_seconds
        )
```

---

## Security Review Checklist

For every service before production:

- [ ] JWT validation on all authenticated endpoints
- [ ] Tenant scope enforcement on all data queries (tenant_id filter)
- [ ] Input sanitization before AI model calls
- [ ] Output sanitization after AI model responses
- [ ] Secrets from Vault only (grep for `os.environ`, `config.get("api_key")`)
- [ ] Rate limiting on all public and tenant-facing endpoints
- [ ] No sensitive data in logs (grep for PII patterns in log output)
- [ ] mTLS configured for service-to-service calls
- [ ] OWASP Top 10 review completed
- [ ] Container image scanned (zero Critical CVEs)
