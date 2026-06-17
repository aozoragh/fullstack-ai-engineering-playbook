# Example Architecture: AI Customer Support Automation

A worked reference architecture for an AI workflow that triages incoming support requests, drafts responses grounded in a knowledge base, and **routes sensitive actions through human approval**. The defining constraint here is *side effects*: this system can email customers and modify tickets, so correctness and approval gates matter more than raw answer quality.

**Example stack (illustrative, not prescriptive):** an API plus a queue and workers for runs (e.g. Postgres-backed jobs or a managed queue); a state store for the run state machine; the same retrieval stack as the RAG chatbot for help-center search; an action layer wrapping outbound integrations (email, ticketing) behind an allowlist; a hosted LLM provider.

---

## Use case

- Incoming support messages (email, form, chat) arrive as tickets.
- The system classifies intent/priority, drafts a grounded reply (RAG over the help center), and either:
  - **auto-sends** low-risk replies (FAQ-style), or
  - **queues for human approval** anything sensitive (refunds, account changes, legal, angry/churn-risk).
- Everything is logged, metered, and idempotent so retries can't double-send.

**Non-goals:** fully autonomous account changes; replacing human agents for sensitive cases.

---

## Core flow

1. **Ingest:** ticket created → normalized → workflow run enqueued (idempotent on ticket id).
2. **Classify:** model assigns intent, priority, and a **risk level** (drives the approval gate).
3. **Retrieve:** RAG over help-center docs, tenant-filtered, threshold-gated.
4. **Draft:** model drafts a reply grounded in retrieved articles, with citations; output schema-validated.
5. **Gate:** 
   - low risk + high confidence → auto-send;
   - otherwise → **human approval** queue with the drafted action previewed.
6. **Act:** on approval (or auto), perform the side effect (send reply / update ticket) — **exactly once**.
7. **Record:** usage, outcome, approver, and the full run trace.

See [`../diagrams/ai-workflow-builder.md`](../diagrams/ai-workflow-builder.md).

---

## Main components

| Component | Responsibility |
| --- | --- |
| Intake API / connector | Receive tickets (webhook/email), normalize, enqueue run |
| Orchestrator | Drive the run through steps; enforce step/iteration cap |
| Classifier step | Intent, priority, risk level (cheap model) |
| Retrieval (RAG) | Tenant-filtered help-center search + threshold |
| Drafting step | Grounded reply + citations (stronger model), schema-validated |
| Risk/approval gate | Decide auto vs. human; build action preview |
| Approval UI/API | Human reviews, edits, approves/rejects |
| Action executor | Idempotent side effects (send/update), allowlisted |
| Audit & usage log | Who/what/when, tokens, cost, outcome |

---

## API / data considerations

- **Resources:** `workflows/{id}/runs`, `runs/{run_id}` (status/steps/outputs), `runs/{run_id}/approve`, `runs/{run_id}/cancel`, `usage` (see [`../docs/07-api-design-patterns.md`](../docs/07-api-design-patterns.md)).
- **Idempotency:** runs keyed on ticket id; the action executor dedupes so a retry never double-sends.
- **Async + webhooks:** runs are async; `run.needs_approval` and `run.completed` events notify operators/systems (signed, idempotent receivers).
- **State machine:** `queued → classifying → drafting → awaiting_approval → executing → done/failed/cancelled`; transitions persisted and observable.
- **Isolation:** every run, retrieval, and action scoped to the tenant; allowlist of permitted actions per workspace.
- **Errors:** structured; partial failures leave the run in a safe, retryable state.

---

## Failure modes

| Failure | Symptom | Mitigation |
| --- | --- | --- |
| Double-send on retry | Customer emailed twice | Idempotency key + executor dedupe |
| Injection via ticket/doc | AI takes attacker's instruction | Untrusted input; risk gate; allowlisted actions |
| Auto-send of sensitive action | Refund/legal sent without review | Risk classification → mandatory approval gate |
| Misclassification | Wrong routing/priority | Confidence threshold → human; eval on classifier |
| Weak retrieval | Ungrounded/incorrect reply | Threshold → escalate to human, don't fabricate |
| Provider error mid-run | Stuck/partial run | Bounded retry; safe-state persistence; resume |
| Runaway loop | Cost/latency spike | Hard step/iteration cap |
| Quota exceeded | Silent drops | Fail safe with clear status; alert |

---

## Production considerations

- **Human-in-the-loop is a hard guardrail, not a suggestion** — sensitive actions cannot execute without logged approval and an action preview.
- **Idempotency everywhere with side effects** — runs and the executor both dedupe.
- **Observability:** per-run trace (each step's input/output/tokens/latency), approval audit, auto-vs-human ratio, cost per run.
- **Cost control:** cheap model for classification, stronger model only for drafting; bounded context; step cap prevents recursion.
- **Security:** action executor restricted to an allowlist; retrieved/ticket content untrusted; secrets server-side.
- **Rollout:** new prompts/classifier versions behind the eval gate; canary on a fraction of tickets; rollback via config.

---

## Testing strategy

- **Unit:** classifier output schema; risk-level mapping; idempotency key handling; step-cap enforcement.
- **Integration (provider mocked):** full run state machine; retry leaves safe state; executor dedupes side effects.
- **E2E:** ticket → draft → approval → send, through real API + auth; rejected draft is not sent.
- **Workflow eval:** representative tickets → correct intent/priority/routing; sensitive cases always route to approval.
- **Refusal/safety:** injection in a ticket can't trigger an unapproved action or non-allowlisted call.
- **Cost/latency:** bounded tokens per run; mocked-call count proves no runaway fan-out.

See [`../docs/08-testing-strategy.md`](../docs/08-testing-strategy.md) and [`../docs/05-ai-evaluation-checklist.md`](../docs/05-ai-evaluation-checklist.md).

---

## Possible future improvements

1. **Confidence-calibrated auto-send thresholds** tuned from human-approval outcomes.
2. **Feedback loop:** approver edits become training signal for prompt/classifier improvement (and new golden cases).
3. **Multi-channel actions** (chat, messaging) behind the same allowlist + approval model.
4. **SLA-aware prioritization** and escalation when approval queues back up.
5. **Per-action simulation/preview** for higher-risk operations before approval.
