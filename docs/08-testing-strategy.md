# Testing Strategy for AI Products

A standard test pyramid still applies — but AI features add non-determinism, expensive external calls, and failure modes (hallucination, weak-context, injection) that ordinary tests don't cover. This strategy keeps the deterministic parts deterministic and adds a dedicated layer for AI-specific regressions.

> Principle: **mock the provider in the test suite; evaluate the model in the eval suite.** Don't make CI flaky or expensive by calling live models for unit/integration tests, and don't pretend a passing unit test proves answer quality.

---

## Test layers

### Unit tests
Fast, deterministic, no network.
- [ ] Chunking: boundaries, overlap, metadata attached correctly, deterministic output.
- [ ] Prompt assembly: correct prompt ID/version, untrusted content placed in the right region, context budget respected.
- [ ] Output parsing/validation: valid schema passes; malformed output is rejected, not best-guessed.
- [ ] Token/context budgeting math, truncation, and history-window logic.
- [ ] Pure business logic around the AI call (quota checks, idempotency key handling).

### Integration tests
Components together, **provider mocked** with recorded/stubbed responses.
- [ ] Ingestion pipeline: upload → extract → chunk → embed (mocked) → store; idempotent re-ingest.
- [ ] Retrieval: tenant filter applied, threshold enforced, `top_k` respected, scores returned.
- [ ] End-to-end answer path with a stubbed model: question → retrieval → prompt → parsed answer + sources.
- [ ] Error handling: provider timeout, `429`, `5xx` → bounded retry → graceful fallback.
- [ ] Async jobs/webhooks: enqueue → process → status transitions → idempotent delivery.

### E2E tests
Full stack through the real API (still mock/stub the LLM provider for determinism).
- [ ] Critical user journeys: sign in → upload doc → ask → see grounded answer with sources.
- [ ] Auth and tenant scoping enforced through the real routes.
- [ ] Rate limits and quotas return correct status codes.
- [ ] Human-approval workflow: sensitive action blocks until approved.

### AI-specific regression tests
Run against the **golden set** (see evaluation checklist). These guard behavior that code-level tests can't.
- [ ] Golden-set score must **not regress** below the current production baseline.
- [ ] Runs on every prompt/model/retrieval change as a release gate.
- [ ] New production bugs are added as golden cases so they can't silently return.

### RAG source-matching tests
- [ ] For each golden question, retrieval surfaces the expected source chunk(s) (recall@k / source-match).
- [ ] Citations in the answer point to chunks that actually support it (groundedness).
- [ ] Cross-tenant isolation: seed two tenants; confirm tenant A's query never returns tenant B's chunks.

### Refusal tests
The most under-tested and most important AI behavior.
- [ ] Out-of-scope question → refusal, not fabrication.
- [ ] Empty / below-threshold retrieval → "not found in your documents," not a guess.
- [ ] Prompt-injection payloads (in user input **and** in uploaded documents) → instructions ignored, no leaked system prompt, no unauthorized tool call.
- [ ] Unsafe/disallowed requests → declined per policy.

### Workflow automation tests
- [ ] Each workflow produces the correct action/extraction/routing for representative inputs.
- [ ] Failure paths: bad input, provider error, partial completion → safe state + retryable.
- [ ] Idempotency: a retried run executes side effects exactly once.
- [ ] Step/iteration cap prevents runaway loops.
- [ ] Approval gate: sensitive steps do not execute without explicit approval.

### Cost & latency checks
- [ ] Token usage per request stays within expected bounds (assert on token counts from the mocked layer).
- [ ] No unbounded fan-out (per-row LLM calls, retry storms) — caught by counting mocked calls.
- [ ] Latency budgets asserted where feasible; p95 tracked in staging.
- [ ] A cost regression on the golden set (tokens up sharply) flags for review.

---

## Example test plan: Document RAG Chatbot

| Layer | Test | Asserts |
| --- | --- | --- |
| Unit | Chunker splits a 30-page PDF | Deterministic chunks, overlap correct, page metadata attached |
| Unit | Prompt builder | Retrieved text in untrusted region; `max_output_tokens` set; prompt id+version stamped |
| Unit | Output validator | Valid JSON w/ `answer` + `sources[]` passes; missing `sources` rejected |
| Integration | Ingest → retrieve (mocked embeddings) | Re-ingest is idempotent; tenant filter applied; threshold drops low scores |
| Integration | Provider timeout | Bounded retry then graceful fallback message; no hang |
| E2E | Upload → ask → answer | Answer returns with correct sources via real API + auth |
| Regression | Golden set (40 Q&A + 10 refuse) | Score ≥ baseline; all 10 refuse cases refuse |
| Source-match | Recall@k on golden set | Expected chunk retrieved for each answerable question |
| Isolation | Two-tenant seed | Tenant A query never returns Tenant B chunks |
| Refusal | Injection payload in uploaded doc | System prompt not leaked; instruction ignored |
| Cost | Per-question token count | Within budget; no unbounded context growth |

---

## Decision rules

- **Mock providers in CI; evaluate models in the eval suite.** Keep CI fast, deterministic, and cheap.
- **Test the refusal and failure paths, not just the happy path** — that's where AI products break.
- **Every production incident becomes a regression test** (code-level) and/or a golden case (behavior-level).
- **Isolation is a test, not a hope** — seed two tenants and prove no leak.
- **Count the calls** — assertions on mocked-call counts catch unbounded fan-out before it bills you.

See also: [`05-ai-evaluation-checklist.md`](05-ai-evaluation-checklist.md), [`01-rag-architecture-checklist.md`](01-rag-architecture-checklist.md), and [`09-github-pr-review-workflow.md`](09-github-pr-review-workflow.md).
