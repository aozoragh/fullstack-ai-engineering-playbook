# AI Feature Spec — <Feature Name>

> Copy this file into your repo (e.g. `specs/<feature>.md`) and fill it in before building. Delete the guidance in quotes. A spec that can't answer "how do we evaluate this?" and "what does it refuse to do?" is not ready to build.

| Field | Value |
| --- | --- |
| Status | Draft / In review / Approved / Shipped |
| Owner | |
| Reviewers | |
| Last updated | YYYY-MM-DD |
| Related docs | spec / design / tickets |

---

## 1. Problem & goal
> What user problem does this solve? What does success look like in one sentence?

-

## 2. Users & use case
> Who uses this, in what context, how often? What's the primary job-to-be-done?

-

## 3. Scope
**In scope**
-

**Out of scope** (explicitly)
-

## 4. User flow
> The happy path, step by step. Note where the AI call happens.

1.
2.
3.

## 5. AI behavior
- **Task type:** (classify / extract / generate / answer-from-context / route / agent)
- **Model & settings:** model id, temperature, max output tokens
- **Prompt:** ID + version (reference, don't inline)
- **Inputs (mark trust level):** user input (untrusted), retrieved docs (untrusted), system context (trusted)
- **Output:** format / schema (link the schema)
- **Grounding:** must answer only from provided context? (Y/N)

## 6. What it must refuse / not do
> The guardrails. Be explicit — this is where AI features fail.

- Refuses when: (e.g. no relevant context, out-of-scope, unsafe request)
- Must never: (e.g. invent facts, reveal system prompt, act without approval)
- Injection handling: retrieved/user content treated as untrusted; cannot trigger sensitive actions.

## 7. Data
- **Data in:** what data is sent to the model? Any PII? Can it be minimized/redacted?
- **Data stored:** documents, embeddings, history, usage — where, with what retention?
- **Isolation:** how is per-tenant/user isolation enforced (data layer)?
- **Deletion:** how do deletes propagate to vectors/caches/logs?

## 8. API surface
> Endpoints, request/response shape, errors. Note idempotency and async jobs.

-

## 9. Evaluation plan
> How do we know it works — before and after shipping?

- **Golden set:** location, size, includes refusal cases? (Y/N)
- **Metrics:** (accuracy / recall@k / source-match / groundedness / refusal correctness)
- **Baseline & gate:** must beat <baseline> on the golden set to ship.
- **Human review:** what's sampled and by whom?

## 10. Cost & limits
- **Estimated tokens/cost per call** and **per user/run:**
- **Bounds:** max context, `top_k`, `max_output_tokens`, history truncation
- **Quotas / rate limits:**
- **Worst-case cost** (and what prevents unbounded fan-out):

## 11. Failure modes & fallback
| Failure | User-facing behavior | Mitigation |
| --- | --- | --- |
| Provider timeout/down | | timeout + bounded retry + fallback |
| Weak/empty retrieval | | refuse honestly |
| Malformed model output | | schema-validate + reject |
| Quota exceeded | | clear 429, fail safe |

## 12. Observability
- Logged per call: request id, tenant, prompt id+version, tokens, latency, outcome (PII-redacted)
- Metrics/alerts: error rate, latency p95, token/cost anomalies

## 13. Rollout & rollback
- Rollout: flagged / canary → full
- Rollback: code + prompt/model/retrieval (independently reversible)

## 14. Open questions
-

---

### Definition of ready (to build)
- [ ] Scope, refusal behavior, and isolation defined
- [ ] Evaluation plan + golden set identified
- [ ] Cost bounds and worst-case stated
- [ ] Failure/fallback behavior specified
- [ ] Rollback plan exists
