# API Design Patterns for AI Products

AI products add resources and constraints that ordinary CRUD APIs don't have: expensive non-deterministic operations, long-running jobs, usage that must be metered, and content that must be tenant-isolated. This document covers the resource model and the patterns that keep such an API predictable, safe, and affordable.

Conventions assumed below: REST-ish JSON, resource-oriented URLs, tenant scoping enforced server-side on **every** route.

---

## Core resources

A typical RAG + workflow product exposes roughly these resources. Keep them consistent and tenant-scoped.

### Documents
The source material users upload/manage.
```
POST   /documents                 # create (upload) → returns {id, status: "processing"}
GET    /documents?cursor=&limit=   # list (paginated, tenant-scoped)
GET    /documents/{id}             # status, metadata, ingestion result
DELETE /documents/{id}             # removes doc AND its chunks/vectors
```
- Ingestion is async — `POST` returns immediately with a `processing` status; the client polls or subscribes (webhook).

### Chunks
The retrievable units derived from documents. Usually internal/admin, sometimes read-only to clients for debugging.
```
GET /documents/{id}/chunks?cursor=&limit=   # inspect chunking/metadata
```

### Chat sessions
A conversation context.
```
POST /chat/sessions                 # create
GET  /chat/sessions?cursor=&limit=  # list
GET  /chat/sessions/{id}            # metadata + truncated history
DELETE /chat/sessions/{id}
```

### Messages
Turns within a session; this is where the AI call happens.
```
POST /chat/sessions/{id}/messages   # ask a question → grounded answer + sources[]
GET  /chat/sessions/{id}/messages?cursor=&limit=
```
- Response includes `sources` (document + section) and a flag when the system **refused** due to weak context.

### Workflow runs
An execution of an automation/agent pipeline.
```
POST /workflows/{id}/runs           # start a run (idempotent) → {run_id, status: "queued"}
GET  /workflows/{id}/runs?cursor=&limit=
GET  /workflows/{id}/runs/{run_id}  # status, steps, outputs, approvals pending
POST /workflows/{id}/runs/{run_id}/approve   # human approval gate
POST /workflows/{id}/runs/{run_id}/cancel
```

### AI usage tracking
Metered usage for cost, quotas, and billing.
```
GET /usage?from=&to=&group_by=workspace|user|feature
```
- Backed by per-call usage records (model, tokens in/out, cost, latency, outcome).

### Admin knowledge base
Admin-managed shared/global content (separate from per-user docs).
```
POST   /admin/kb/sources            # add a managed source
GET    /admin/kb/sources
DELETE /admin/kb/sources/{id}
POST   /admin/kb/reindex            # triggered re-embedding job
```
- Admin routes require elevated role; actions are audited.

---

## Patterns

### Idempotency for expensive AI actions
Expensive/irreversible operations (generation, ingestion, workflow runs, anything that costs money or sends something) must be idempotent so retries don't double-charge or double-execute.
- Accept an `Idempotency-Key` header on `POST`s that trigger AI work.
- Store key → result; a repeat key returns the original result instead of re-running.
- Keys are scoped per tenant and expire after a sensible window.
```
POST /workflows/{id}/runs
Idempotency-Key: 7c1f...-client-generated
```

### Pagination
- **Cursor-based** pagination on all list endpoints (stable under inserts; scales better than offset).
- Bounded, default `limit`; never return unbounded lists.
- Responses include `next_cursor` (null when exhausted).
```json
{ "data": [ /* ... */ ], "next_cursor": "eyJpZCI6..." }
```

### Structured errors
- Consistent, machine-readable error shape across the whole API.
- Stable `code`s clients can switch on; human-readable `message`; `request_id` for support.
- Never leak stack traces, secrets, internal IDs, or prompts in errors.
```json
{
  "error": {
    "code": "context_too_weak",
    "message": "No relevant content found in your documents for this question.",
    "request_id": "req_8f2a...",
    "retryable": false
  }
}
```
- Use the right status: `400` validation, `401/403` auth/authorization, `404` not found, `409` conflict/idempotency, `422` schema, `429` rate/quota, `503` provider down.

### Async jobs
Long-running AI work (ingestion, batch embedding, long generations, workflow runs) must not block a request.
- `POST` enqueues and returns `202` (or `201` + `status: queued/processing`) with a resource to poll.
- Job resource exposes `status`, `progress`, `result`, and `error`.
- Clients poll `GET` or receive a webhook on completion.

### Webhooks
For notifying clients/systems of async completion and events.
- Signed payloads (HMAC) so receivers can verify authenticity.
- At-least-once delivery → receivers must be idempotent (include an event ID).
- Retries with backoff on delivery failure; a dead-letter path for giving up.
- Documented event types (`document.ingested`, `run.completed`, `run.needs_approval`, `usage.threshold_reached`).

### Access control
- Every route authorizes against the caller's tenant/role **server-side** — never trust client-supplied tenant IDs.
- Resource lookups are always tenant-scoped (`WHERE workspace_id = :caller_ws`), so an ID from another tenant returns `404`, not someone else's data.
- Admin/KB and approval routes require elevated roles; sensitive actions are audited (who/when/what).
- Object-level checks on every read/write (not just route-level) to prevent IDOR.

---

## Decision rules

- **Expensive/irreversible → idempotent.** Always assume the client will retry.
- **Long-running → async job + poll/webhook.** Never hold a request open on a model call you don't control.
- **List → cursor-paginated and bounded.** No unbounded responses.
- **Errors are structured, safe, and actionable** — code + message + request_id, never internals.
- **Authorize server-side, per object, per tenant** — the URL is not an authorization.

See also: [`02-llm-app-security-checklist.md`](02-llm-app-security-checklist.md) (isolation, output safety), [`04-cost-and-token-optimization.md`](04-cost-and-token-optimization.md) (usage tracking), and [`../examples/ai-saas-mvp-architecture.md`](../examples/ai-saas-mvp-architecture.md).
