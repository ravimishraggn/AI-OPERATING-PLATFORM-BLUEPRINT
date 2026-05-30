# API Standards — Enterprise AI Operating Platform

> **Document Type:** Platform Standards
> **Status:** Accepted
> **Owner:** Platform Architecture Team
> **Last Updated:** 2026-05-30

---

## Purpose

All platform APIs must conform to these standards. APIs that do not conform will not pass the platform architecture review.

---

## 1. URL Design

```
Base URL structure:
  https://{environment}.platform.internal/api/v{version}/{resource}

Examples:
  https://prod.platform.internal/api/v1/agents
  https://staging.platform.internal/api/v1/models/invoke
  https://prod.platform.internal/api/v2/agents/{id}/runs
```

**Rules:**
- Use plural nouns for collections: `/agents`, `/models`, `/tenants`
- Use kebab-case for multi-word resources: `/agent-runs`, `/model-invocations`
- No verbs in URLs (use HTTP methods for actions): `/agents/{id}/runs` not `/agents/{id}/run`
- Exception: Actions on resources: `/agents/{id}/runs/{run_id}/pause`
- Version in URL path: `/api/v1/`, `/api/v2/`

---

## 2. HTTP Methods

| Method | Purpose | Body | Idempotent |
|---|---|---|---|
| GET | Retrieve resource(s) | No | Yes |
| POST | Create resource or trigger action | Yes | No |
| PUT | Replace resource | Yes | Yes |
| PATCH | Partial update | Yes | No |
| DELETE | Remove resource | No | Yes |

---

## 3. Request and Response Format

**All requests and responses use JSON.**

Required response structure:
```json
// Success response (single resource)
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-05-30T14:23:00.000Z"
  }
}

// Success response (collection)
{
  "data": [ ... ],
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-05-30T14:23:00.000Z",
    "pagination": {
      "total": 150,
      "page": 1,
      "page_size": 25,
      "has_next": true
    }
  }
}

// Error response
{
  "error": {
    "code": "AGENT_NOT_FOUND",
    "message": "Agent 'loan-underwriting-agent' not found in tenant 'tenant-bankA'",
    "details": { "agent_id": "loan-underwriting-agent" },
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
  },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-05-30T14:23:00.000Z"
  }
}
```

---

## 4. HTTP Status Codes

| Code | Use Case |
|---|---|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST (resource created) |
| 202 Accepted | Async operation started |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Validation error, malformed request |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource not found |
| 409 Conflict | Resource already exists, state conflict |
| 422 Unprocessable Entity | Business rule violation |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unexpected server error |
| 503 Service Unavailable | Dependency unavailable |

---

## 5. Pagination

```
Query parameters:
  ?page=1&page_size=25   (page-based, default)
  ?cursor=abc123          (cursor-based, for large collections)
  ?limit=25&offset=0      (offset-based, for simple cases)

Default page size: 25
Maximum page size: 100
```

---

## 6. Filtering, Sorting, and Search

```
Filtering:   ?status=active&tenant_id=bankA
Sorting:     ?sort=created_at&order=desc
Search:      ?q=loan+underwriting
Date range:  ?from=2026-01-01&to=2026-05-30
```

---

## 7. Required Headers

```
# All requests:
Authorization: Bearer {jwt_token}
Content-Type: application/json
X-Request-ID: {client-generated-uuid}  (for idempotency and debugging)
X-Tenant-ID: {tenant_id}               (redundant with JWT; for routing efficiency)

# For mutable operations (optional but recommended):
X-Idempotency-Key: {client-generated-uuid}  (prevents duplicate operations)
```

---

## 8. Versioning Policy

```
API Versioning:
  - Major version in URL path (/api/v1/, /api/v2/)
  - Minor versions (non-breaking additions) do not change the URL
  - Breaking changes require a new major version
  - Old version maintained for 6 months after new version release
  - Deprecation header: "Deprecation: true; Sunset: 2027-06-01"
```

---

## 9. OpenAPI Specification

**Every API must have an OpenAPI 3.1 specification.**

Required fields in OpenAPI spec:
- `info.title`, `info.version`, `info.description`
- `info.contact` (platform team)
- `servers` (environments)
- All endpoints with `operationId`, `summary`, `description`
- All request/response schemas with `description` on each field
- Security requirements (`securitySchemes`, `security`)
- Error responses documented (400, 401, 403, 404, 429, 500)

---

## 10. Rate Limiting Response

When rate limited (429):
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded for tenant 'tenant-bankA' on resource 'model-invocations'",
    "details": {
      "limit": 1000,
      "remaining": 0,
      "reset_at": "2026-05-30T14:24:00.000Z"
    }
  }
}
```

Response headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1717027440
Retry-After: 60
```

---

## 11. Async Operations

For long-running operations, return 202 Accepted with a status URL:

```json
// Response to POST /agents/{id}/runs
{
  "data": {
    "run_id": "run-uuid",
    "status": "pending",
    "status_url": "/api/v1/agents/runs/run-uuid",
    "stream_url": "/api/v1/agents/runs/run-uuid/stream"
  }
}
```

---

## 12. Sensitive Data in APIs

- Never return raw API keys, passwords, or secrets in responses
- Hash or mask credentials before returning (show only last 4 characters)
- PII in responses must be authorized by requester's permissions
- Audit log every API call that returns sensitive data
