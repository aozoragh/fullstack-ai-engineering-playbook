# Deployment Checklist — <Release / Feature>

> Walk this before every production deploy of an AI feature. Copy into the release ticket. See [`docs/06-deployment-readiness-checklist.md`](../docs/06-deployment-readiness-checklist.md).

| Field | Value |
| --- | --- |
| Release / version | |
| Deployer | |
| Date | YYYY-MM-DD |
| Change type | feature / fix / prompt / model / retrieval / schema |

---

## Pre-deploy

### Config & secrets
- [ ] All required env vars set in target environment.
- [ ] Config validated at startup (app fails fast if missing).
- [ ] `.env.example` updated with any new variables (placeholders only).
- [ ] No secrets in client bundle, logs, or git; secret scan clean.

### Data
- [ ] Migrations reviewed, reversible, and tested on a prod-like copy.
- [ ] Backward-compatible during rollout (expand → migrate → contract).
- [ ] Required indexes added (verified with a query plan).
- [ ] Large/locking migrations scheduled safely.

### Resilience
- [ ] Timeouts on every external call (LLM, embeddings, vector store, 3rd-party).
- [ ] Bounded retries (backoff + jitter) on retryable errors only.
- [ ] Circuit breaker / spend cap armed.
- [ ] Graceful fallback when provider is slow/down (no hang, honest message).
- [ ] Idempotency on expensive/irreversible actions.

### AI-specific (if prompt/model/retrieval changed)
- [ ] Eval gate passed: golden-set score ≥ baseline, refusal cases pass.
- [ ] Cost/latency delta within budget.
- [ ] Prompt/model/retrieval version recorded and independently rollback-able.
- [ ] Re-index completed/planned (for chunking/embedding changes).

### Observability
- [ ] Logs include request id, tenant, prompt id+version, tokens, latency, outcome (PII-redacted).
- [ ] Metrics live: error rate, latency p50/p95/p99, token usage, cost.
- [ ] Alerts armed: error spikes, latency regression, cost/usage anomaly.

---

## Deploy
- [ ] Staged rollout (canary / % traffic) before full.
- [ ] Rollback triggers defined: error rate > __%, latency p95 > __ms, cost anomaly, eval drop.
- [ ] On-call / deployer watching dashboards during rollout.

## Post-deploy
- [ ] Smoke tests pass against live env (ingest → ask → grounded answer → refusal case).
- [ ] Health/readiness checks green (DB, vector store, provider reachable).
- [ ] Error rate, latency, and cost nominal for __ minutes.
- [ ] Key user journey verified manually.

## Rollback plan
- [ ] Code rollback: one-step, previous build redeployable.
- [ ] Prompt/model/retrieval rollback: config/flag to last known-good version.
- [ ] Migration: tested down-path or forward-fix documented.
- [ ] Owner + comms plan if rollback is triggered.

---
**Go / No-go:** ____  ·  **Decided by:** ____  ·  **Time:** ____
