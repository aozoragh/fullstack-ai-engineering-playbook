# Deployment Readiness Checklist

What to confirm before an AI feature goes to production — and what must exist so that when it misbehaves (it will), you can see it and recover. This is the checklist I walk before shipping and the one I expect a reviewer to hold me to.

> Treat an AI feature as production-ready only when it is observable, bounded, and reversible. Working once in a demo is not the bar.

---

## 1. Environment & configuration

- [ ] All config comes from environment variables / a secret manager — nothing hardcoded.
- [ ] Required config is validated at startup; the app **fails fast** with a clear message if something is missing, rather than failing mysteriously at first request.
- [ ] Config differs cleanly per environment (dev/staging/prod) with no prod secrets in lower environments.

## 2. `.env.example`

- [ ] An up-to-date `.env.example` lists **every** required variable with a description and a safe placeholder.
- [ ] No real secrets in the example file (placeholders only).
- [ ] New config added in this change is reflected in `.env.example` (reviewers check this).

```dotenv
# .env.example
LLM_API_KEY=                 # server-side only; provider key
LLM_MODEL=claude-opus-4-8    # pinned model id
EMBEDDING_MODEL=             # pinned embedding model id
VECTOR_DB_URL=               # vector store connection
DATABASE_URL=                # primary DB connection
MAX_UPLOAD_MB=20             # file upload limit
RATE_LIMIT_PER_MIN=60        # per-user request limit
REQUEST_TIMEOUT_MS=30000     # provider call timeout
```

## 3. Server-side secrets

- [ ] No provider keys, DB credentials, or signing secrets reachable from the client bundle.
- [ ] Secret scanning runs in CI; `.env` and credential files are gitignored.
- [ ] Secrets are rotatable without a code change; rotation procedure is documented.

## 4. Database migrations

- [ ] Schema changes ship as **versioned, reversible migrations** — never manual prod edits.
- [ ] Migrations are tested on a prod-like copy and are backward-compatible during rollout (expand → migrate → contract).
- [ ] Long/locking migrations on large tables are planned (batched, off-peak) so they don't cause downtime.

## 5. Indexes

- [ ] Indexes exist for tenant-scoping fields, foreign keys, and hot query paths (lists, lookups, pagination).
- [ ] Vector index parameters match the embedding model (metric, dimensions).
- [ ] New query patterns introduced by this change have supporting indexes — verified with a query plan, not assumed.

## 6. Model & provider lifecycle

- [ ] Model **and** embedding versions are pinned (not "latest") so behavior doesn't drift under you.
- [ ] You track provider deprecation notices — vendors retire model snapshots on a schedule, and a forced migration mid-incident is the worst time to discover it.
- [ ] A model/provider change has a migration path: re-run the eval gate, plan re-embedding if the embedding model changes, and keep the prior version rollback-able.
- [ ] Where the product warrants it, a secondary provider/model is identified so a deprecation or outage isn't a single point of failure.

## 7. Provider timeout / retry handling

- [ ] Every external call (LLM, embeddings, vector store, third-party) has an explicit **timeout**.
- [ ] Retries are bounded with exponential backoff + jitter, only on retryable errors (`429`, `5xx`, timeout).
- [ ] A **circuit breaker** prevents retry storms from amplifying an outage into a cost/latency incident.
- [ ] Idempotency protects expensive/irreversible actions from double execution on retry.

## 8. Fallback behavior

- [ ] When the AI provider is down/slow/rate-limited, the user gets a graceful, honest message — not a hang or a stack trace.
- [ ] Where feasible, a fallback exists (secondary model/provider, cached result, queued-for-later, or degraded-but-safe mode).
- [ ] Fallbacks never compromise isolation or skip security checks to "keep working."

## 9. Observability

- [ ] Structured logs with request ID, workspace ID, model, prompt version, token counts, latency, and outcome (redacted of PII).
- [ ] Metrics: request rate, error rate, latency (p50/p95/p99), token usage, cost — per model/endpoint.
- [ ] Tracing across the request path (retrieval → model → post-processing) for debugging slow/failed calls.
- [ ] Alerts on error-rate spikes, latency regressions, and **cost/usage anomalies**.

## 10. Token / cost monitoring

- [ ] Token usage and spend are tracked per user/workspace/feature and visible on a dashboard.
- [ ] Spend caps / circuit breakers are armed; alerts fire before, not after, a budget blowout.
- [ ] A baseline cost-per-request is known so regressions are detectable.

## 11. Smoke tests

- [ ] Post-deploy smoke tests hit the critical AI paths (ingest a doc, ask a question, get a grounded answer, trigger a refusal) against the live environment.
- [ ] Health/readiness endpoints check downstream dependencies (DB, vector store, provider reachability).
- [ ] Smoke tests run automatically after deploy and block/flag a bad release.

## 12. Rollback plan

- [ ] Code rollback is a one-step, tested operation (previous build redeployable).
- [ ] **Prompt/model/retrieval config can be rolled back independently** of code (flag/config) — the last known-good version is identified.
- [ ] Migrations have a tested down-path or a forward-fix plan; data changes are reversible or recoverable from backup.
- [ ] The rollout is staged (canary → full) with clear rollback triggers (error rate, eval score, cost).

---

## Pre-deploy gate (all must be true)

| Area | Ready? |
| --- | --- |
| Config validated + `.env.example` current | ☐ |
| Secrets server-side, scanned | ☐ |
| Migrations reversible + indexes in place | ☐ |
| Models/embeddings pinned; deprecation tracked | ☐ |
| Timeouts, bounded retries, circuit breaker | ☐ |
| Fallback for provider failure | ☐ |
| Logs / metrics / cost monitoring live | ☐ |
| Smoke tests pass against target env | ☐ |
| Eval gate passed (prompt/model/retrieval changes) | ☐ |
| Rollback plan written + tested | ☐ |

## Decision rules

- **Fail fast on bad config**, fail gracefully on provider errors.
- **Everything reversible**: code, prompt, model, retrieval, schema.
- **No feature ships without observability** for its own cost and errors.
- **Stage the rollout** with explicit rollback triggers — don't flip 100% blind.

See also: [`../templates/deployment-checklist.md`](../templates/deployment-checklist.md), [`04-cost-and-token-optimization.md`](04-cost-and-token-optimization.md), and [`../diagrams/deployment-flow.md`](../diagrams/deployment-flow.md).
