---
type: Spec
title: "AgentForge API Specification v0.1.0"
description: REST API and WebSocket specification for the AgentForge cloud platform. Covers all endpoints for pipeline management, compilation, execution, event streaming, billing, and team management. Version 0.1.0.
tags: [spec, api, rest, websocket, endpoints, billing, auth]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge API Specification v0.1.0

**Base URL:** `https://api.agentforge.dev/v1`

All requests require authentication via `Authorization: Bearer <jwt_token>` unless noted.

---

## 1. Authentication

### POST /auth/signup
Creates a new user account.

**Request:**
```json
{ "email": "user@example.com", "password": "...", "name": "Jane Developer" }
```

**Response 201:**
```json
{ "user_id": "uuid", "token": "jwt", "expires_at": "ISO8601" }
```

### POST /auth/login
Authenticates an existing user.

**Request:** `{ "email": "...", "password": "..." }`
**Response 200:** `{ "token": "jwt", "expires_at": "ISO8601" }`

### POST /auth/refresh
Refreshes a JWT token (within 7 days of expiry).

**Response 200:** `{ "token": "jwt", "expires_at": "ISO8601" }`

---

## 2. Pipeline Management

### GET /pipelines
Lists pipelines owned by the authenticated user (or team, if `team_id` header set).

**Query params:** `page`, `per_page` (default 20), `search`

**Response 200:**
```json
{
  "pipelines": [
    {
      "id": "uuid",
      "name": "RefactorDeadCode",
      "created_at": "ISO8601",
      "updated_at": "ISO8601",
      "last_run_at": "ISO8601",
      "visibility": "private | team | public"
    }
  ],
  "total": 42,
  "page": 1
}
```

### POST /pipelines
Creates a new pipeline.

**Request:**
```json
{ "name": "MyPipeline", "source": "<.agent file content>", "visibility": "private" }
```

**Response 201:**
```json
{ "id": "uuid", "name": "MyPipeline", "created_at": "ISO8601" }
```

### GET /pipelines/{id}
Returns a single pipeline with its source.

**Response 200:**
```json
{
  "id": "uuid", "name": "MyPipeline",
  "source": "<.agent content>", "visibility": "private",
  "created_at": "ISO8601", "updated_at": "ISO8601"
}
```

### PUT /pipelines/{id}
Updates pipeline source or visibility.

**Request:** `{ "source": "...", "visibility": "team" }`
**Response 200:** Updated pipeline object.

### DELETE /pipelines/{id}
Deletes a pipeline and all its run history.

**Response 204:** No content.

---

## 3. Compilation

### POST /compile
Compiles a `.agent` source string and returns the IR or errors.

**Request:**
```json
{ "source": "<.agent file content>", "pipeline_id": "uuid (optional)" }
```

**Response 200 (success):**
```json
{
  "ok": true,
  "ir": { "version": "1.0", "pipeline": "...", "instructions": [...] },
  "warnings": [
    { "code": "W001", "message": "Back-edge detected: Validate → FindDeadCode", "line": 42 }
  ]
}
```

**Response 400 (compile error):**
```json
{
  "ok": false,
  "errors": [
    { "code": "E030", "message": "Type mismatch: expected FileContent, got FileList", "line": 18, "col": 14 }
  ],
  "warnings": []
}
```

---

## 4. Run Execution

### POST /runs
Compiles and starts a pipeline execution.

**Request:**
```json
{
  "pipeline_id": "uuid",
  "source": "<optional override source>",
  "dry_run": false,
  "from_step": "optional step name to resume from",
  "model_overrides": { "default": "gpt-4o" }
}
```

**Response 202:**
```json
{ "run_id": "uuid", "status": "running", "started_at": "ISO8601" }
```

### GET /runs/{id}
Returns the current state of a run.

**Response 200:**
```json
{
  "run_id": "uuid",
  "pipeline_id": "uuid",
  "status": "running | paused | completed | failed",
  "started_at": "ISO8601",
  "completed_at": "ISO8601",
  "current_step": "StepName",
  "total_tokens": 18240,
  "total_cost_usd": 0.047,
  "events_count": 23
}
```

### GET /runs/{id}/events
Returns all events for a run (completed or in-progress).

**Query params:** `from_index` (default 0), `limit` (default 100)

**Response 200:**
```json
{
  "events": [
    { "index": 0, "event": "RunStarted", "timestamp": "ISO8601", "data": { ... } }
  ],
  "total": 156
}
```

### GET /runs/{id}/stream
Server-Sent Events stream. Streams events in real-time for a running pipeline.

**Response:** `text/event-stream` (SSE)

Each event is NDJSON:
```
data: {"event": "LLMCallCompleted", "run_id": "uuid", "timestamp": "...", "data": {...}}
```

Stream closes with a `RunCompleted` or `RunFailed` event.

### POST /runs/{id}/resume
Resumes a paused run (after a `human.checkpoint`).

**Request:** `{ "action": "approved | rejected", "note": "optional" }`
**Response 200:** `{ "status": "running" }`

### POST /runs/{id}/cancel
Cancels a running or paused run.

**Response 200:** `{ "status": "failed", "message": "Cancelled by user" }`

### GET /runs
Lists recent runs for the authenticated user.

**Query params:** `pipeline_id`, `status`, `from`, `to`, `page`, `per_page`

**Response 200:** Paginated list of run summary objects.

---

## 5. Marketplace

### GET /marketplace/templates
Lists published pipeline templates.

**Query params:** `search`, `tags`, `sort` (`downloads | recent`), `page`, `per_page`

**Response 200:**
```json
{
  "templates": [
    {
      "id": "uuid",
      "name": "Dead Code Finder",
      "description": "...",
      "author": "username",
      "downloads": 1240,
      "tags": ["refactor", "typescript"],
      "created_at": "ISO8601"
    }
  ]
}
```

### GET /marketplace/templates/{id}
Returns a single template including source.

**Response 200:** Template object with `source` field.

### POST /marketplace/templates/{id}/fork
Creates a private pipeline from a marketplace template.

**Response 201:** New pipeline object.

### POST /marketplace/templates
Submits a pipeline for marketplace review. Requires `pipeline_id` of an existing pipeline.

**Request:** `{ "pipeline_id": "uuid", "description": "...", "tags": ["refactor"] }`
**Response 202:** `{ "submission_id": "uuid", "status": "pending_review" }`

---

## 6. Billing

### GET /billing/usage
Returns current month's usage for the authenticated user.

**Response 200:**
```json
{
  "plan": "builder",
  "period_start": "ISO8601",
  "period_end": "ISO8601",
  "executions_used": 342,
  "executions_limit": 5000,
  "bedrock_tokens_used": 1840000,
  "bedrock_cost_usd": 8.47,
  "gpu_tokens_used": 220000,
  "gpu_cost_usd": 0.022,
  "subscription_usd": 29.00
}
```

### GET /billing/invoices
Returns historical invoices.

**Response 200:** Paginated list of invoice objects with `stripe_invoice_id`, `amount_usd`, `period`, `status`.

### POST /billing/portal
Returns a Stripe Customer Portal URL for managing subscription and payment methods.

**Response 200:** `{ "url": "https://billing.stripe.com/..." }`

---

## 7. Teams (Phase 3)

### POST /teams
Creates a new team.

**Request:** `{ "name": "Acme Engineering" }`
**Response 201:** `{ "team_id": "uuid", "name": "...", "created_at": "ISO8601" }`

### GET /teams/{id}/members
Lists team members.

**Response 200:** `{ "members": [{ "user_id": "uuid", "email": "...", "role": "owner | admin | member" }] }`

### POST /teams/{id}/invites
Invites a user to the team.

**Request:** `{ "email": "...", "role": "member" }`
**Response 202:** `{ "invite_id": "uuid" }`

### DELETE /teams/{id}/members/{user_id}
Removes a member from the team.

**Response 204:** No content.

---

## 8. Error Responses

All error responses use the following shape:

```json
{ "error": { "code": "ERROR_CODE", "message": "Human-readable message", "details": { ... } } }
```

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (validation error) |
| 401 | Authentication required or token expired |
| 403 | Forbidden (insufficient permissions) |
| 404 | Resource not found |
| 409 | Conflict (duplicate name, etc.) |
| 429 | Rate limit exceeded |
| 500 | Internal server error |
| 503 | Service temporarily unavailable |

---

## 9. Rate Limits

| Endpoint Group | Limit |
|---|---|
| `POST /compile` | 60 requests/minute |
| `POST /runs` | 20 requests/minute |
| `GET /runs/{id}/stream` | 10 concurrent SSE connections per user |
| All other endpoints | 120 requests/minute |

Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

---

## Related

- [Runtime Spec](runtime-spec.md) — the execution model behind `/runs`
- [Data Model](../06-design/data-model.md) — the database schema behind these endpoints
- [Billing Design](../06-design/billing-design.md) — how billing endpoints work internally
