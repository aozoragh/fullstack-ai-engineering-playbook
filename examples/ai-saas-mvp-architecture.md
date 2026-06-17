# Example Architecture: AI SaaS MVP Starter

A worked reference architecture for a **multi-tenant AI SaaS MVP**: auth, billing, usage limits, and one AI feature wired end-to-end — built so it can grow without rewriting the foundations. The goal of an MVP is to be *small but not fragile*: ship fast, but get isolation, metering, and cost bounds right from day one because they're expensive to retrofit.

**Example stack (illustrative, not prescriptive):** a single web app (e.g. Next.js, or a Node/Python API with an SPA); Postgres with tenant-scoped rows (and row-level security where it fits); an auth library or provider; a billing provider for plans and entitlements; the AI feature behind a server-side service; logs, metrics, and cost data to a hosted observability tool. A monolith with clean module boundaries beats microservices at this stage.

---

## Use case

- A SaaS product where teams sign up, pick a plan, and use an AI feature (e.g. document Q&A or content generation).
- Multi-tenant from the start: organizations/workspaces, users, roles.
- Usage is metered and tied to plan limits; the AI feature is cost-bounded and observable.
- Designed to add more AI features later without re-architecting auth, billing, or isolation.

**Non-goals:** premature scaling, microservices, or features beyond validating the core value.

---

## Core flow

1. **Sign up / onboard:** user creates an org (workspace), invites teammates, picks a plan.
2. **Auth:** sessions/JWT; every request carries the tenant + role; authorized server-side.
3. **Use AI feature:** request → quota check → AI call (bounded) → response → usage recorded.
4. **Meter & bill:** usage records aggregate into plan limits + billing; quotas enforced.
5. **Admin:** workspace admins manage members, plan, and (if applicable) the knowledge base.

---

## Main components

| Component | Responsibility |
| --- | --- |
| Auth & identity | Sign-up/in, sessions, org/workspace, roles |
| Tenant model | Workspace as the isolation boundary on every table |
| Billing | Plans, subscriptions, entitlements (via a payments provider) |
| Quota/usage service | Per-call metering; enforce plan limits; spend caps |
| AI feature service | The end-to-end AI capability (e.g. RAG or generation) |
| Primary DB | Tenant-scoped data; migrations + indexes |
| Admin | Members, roles, plan, KB management |
| Observability | Logs/metrics/cost per tenant + feature |

---

## API / data considerations

- **Tenant scoping is the spine:** `workspace_id` on every table, indexed; every query filters on it; object-level authz prevents IDOR. The MVP's single most important invariant.
- **Roles:** at least `owner/admin/member`; admin and billing routes require elevated roles; sensitive actions audited.
- **Entitlements:** plan → limits (seats, requests, tokens, documents) checked server-side at call time, not in the UI.
- **AI endpoints:** idempotency on expensive actions; async jobs for long work; structured errors (`quota_exceeded`, `plan_limit`); cursor pagination on lists.
- **Usage records:** per call (tenant, user, feature, model, tokens in/out, cost, latency, outcome) → quotas + dashboards + billing (see [`../docs/07-api-design-patterns.md`](../docs/07-api-design-patterns.md)).
- **Secrets:** provider keys server-side only; per-environment config validated at startup.

---

## Failure modes

| Failure | Symptom | Mitigation |
| --- | --- | --- |
| Cross-tenant data access | Org A sees Org B's data | `workspace_id` filter in data layer + isolation test |
| Quota bypass | Cost overrun beyond plan | Server-side entitlement checks + spend caps |
| Billing/usage drift | Charges don't match usage | Usage records as source of truth; reconcile job |
| Unbounded AI cost | Surprise bill | Bounded context/output; per-tenant metering; alerts |
| Provider outage | Feature down hard | Timeout + retry + graceful fallback message |
| Secret in client | Key leak | Server-side only; secret scanning |
| Migration breaks prod | Downtime | Reversible, backward-compatible migrations |

---

## Production considerations

- **Isolation first:** the workspace boundary is enforced in the data layer and covered by an automated two-tenant test — retrofitting isolation later is painful and risky.
- **Meter from day one:** even pre-revenue, record usage per tenant/feature so cost and limits are real, not guessed.
- **Cost bounds on the AI feature:** model selection by task, bounded context/output, caching where safe.
- **Observability per tenant + feature:** error rate, latency, tokens, cost; alerts on anomalies and approaching-limit.
- **Security baseline:** secrets server-side, rate limits, input validation, output safety, audited admin actions.
- **Keep it boring:** a monolith with clean module boundaries beats premature microservices for an MVP; deploy with reversible migrations and a rollback plan.

---

## Testing strategy

- **Unit:** entitlement/quota logic; usage-record math; role checks; idempotency keys.
- **Integration (provider mocked):** AI feature end-to-end; quota enforcement returns correct status; provider error → fallback.
- **E2E:** sign up → invite → use AI feature → see usage update → hit limit, through real API + auth.
- **Isolation:** two-tenant seed; no cross-tenant read/write on any route.
- **AI-specific:** golden set + refusal tests for the AI feature (per [`../docs/08-testing-strategy.md`](../docs/08-testing-strategy.md)).
- **Billing:** plan-limit and entitlement paths; usage reconciles with metered records.

---

## Possible future improvements

1. **Add AI features incrementally** on the same tenant/usage/isolation foundation (the point of getting these right early).
2. **Self-serve usage dashboards** so customers see their own token/cost consumption.
3. **Tiered model routing** per plan (cheaper models on lower tiers).
4. **SSO / SCIM** for enterprise customers as you move upmarket.
5. **Background eval + cost regression in CI** as the AI surface grows.
