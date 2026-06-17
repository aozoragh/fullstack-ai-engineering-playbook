# Example Architecture: AI Document RAG Chatbot

A worked reference architecture for a multi-tenant chatbot that answers questions **strictly from a workspace's uploaded documents**, with citations and honest refusal when the answer isn't in the documents.

This is the canonical RAG product. The hard parts are not "calling the model" — they are isolation, retrieval quality, and refusal.

---

## Use case

- A team uploads documents (policies, contracts, manuals, knowledge base).
- Users ask natural-language questions and get answers **grounded in those documents**, with sources.
- The system **refuses** when the documents don't contain the answer, instead of guessing.
- Each workspace's documents are strictly isolated from every other workspace.

**Non-goals:** general world-knowledge Q&A, web browsing, model fine-tuning.

---

## Core flow

**Ingestion (async)**
1. User uploads a file → validated (type, size) → stored per tenant → ingestion job queued.
2. Worker extracts text → chunks (semantic, with overlap) → attaches metadata (doc id, title, page, tenant id, version).
3. Chunks embedded in batches → upserted into the vector store keyed for idempotency.
4. Status surfaced to the user (`processing` → `ready` / `failed` with reason).

**Query (sync, streamed)**
1. User asks a question in a chat session.
2. Question embedded → vector search **filtered by tenant**, `top_k` with a score threshold.
3. Low-confidence/empty retrieval → **refusal path** ("I couldn't find this in your documents").
4. Otherwise: build prompt (system rules + retrieved context as untrusted reference + question) → call model with bounded output.
5. Validate output against schema → return answer + `sources[]` (doc + section), streamed to the UI.
6. Log usage (tokens, latency, prompt id+version, refused?).

See [`../diagrams/rag-pipeline.md`](../diagrams/rag-pipeline.md).

---

## Main components

| Component | Responsibility |
| --- | --- |
| Upload API | Validate file (type/size), store per tenant, enqueue ingestion |
| Ingestion worker | Extract → chunk → embed → upsert (idempotent) |
| Vector store | Tenant-scoped vectors + metadata; cosine/dot index |
| Primary DB | Documents, chunks metadata, sessions, messages, usage |
| Retrieval service | Embed query, tenant-filtered search, threshold, optional re-rank |
| Answer service | Prompt assembly, model call, schema validation, citations |
| Chat API | Sessions, messages, streaming, refusal handling |
| Usage/metering | Per-call token/cost records → quotas + dashboards |

---

## API / data considerations

- **Resources:** `documents`, `documents/{id}/chunks`, `chat/sessions`, `chat/sessions/{id}/messages`, `usage` (see [`../docs/07-api-design-patterns.md`](../docs/07-api-design-patterns.md)).
- **Async ingestion:** `POST /documents` returns `processing`; client polls or gets a `document.ingested` webhook.
- **Idempotency:** uploads/ingestion keyed so re-upload doesn't duplicate chunks.
- **Pagination:** cursor-based on documents/messages lists.
- **Isolation:** `workspace_id` is a first-class, indexed field on every table and every vector; retrieval filters on it **before** ranking.
- **Deletion:** deleting a document removes its chunks and vectors (and respects data-deletion requests).
- **Errors:** structured (`context_too_weak`, `unsupported_file_type`, `quota_exceeded`) with `request_id`.

---

## Failure modes

| Failure | Symptom | Mitigation |
| --- | --- | --- |
| Empty/low retrieval | Confident wrong answer | Score threshold → refusal path |
| Cross-tenant leak | User sees another workspace's content | Tenant filter in data layer + isolation test |
| Injection via uploaded doc | Model follows doc instructions / leaks prompt | Retrieved text = untrusted; guardrails; refusal test |
| Image-only PDF | Indexed as blank → "not found" everywhere | Detect empty extraction; flag/ OCR |
| Stale index | Answers from deleted/old docs | Re-index on update; delete vectors on delete |
| Provider timeout | Hanging chat | Timeout + bounded retry + fallback message |
| Cost blowup | High bill from big `top_k`/context | Bounded `top_k` + context budget + monitoring |
| Bad chunking | Half-answers | Tuned size/overlap; versioned, re-indexed |

---

## Production considerations

- **Observability:** per query log request id, tenant, top_k, scores, tokens, latency, refused?(bool); alert on retrieval-empty rate and cost anomalies.
- **Cost control:** bound `top_k`, context, and `max_output_tokens`; cache embeddings; summarize long chat history; meter per workspace.
- **Security:** keys server-side; PII minimized before embedding; uploads sandboxed; per-tenant storage paths.
- **Resilience:** circuit breaker on the provider; graceful degraded message; idempotent ingestion.
- **Rollout:** prompt/retrieval config versioned and independently rollback-able; canary new prompts behind the eval gate.

---

## Testing strategy

- **Unit:** chunker determinism + metadata; prompt assembly (untrusted region); output schema validation.
- **Integration (provider mocked):** ingest→retrieve idempotency; tenant filter; threshold; timeout→fallback.
- **E2E:** upload → ask → grounded answer with correct sources, via real API + auth.
- **Regression / golden set:** 40 Q&A + 10 "should refuse"; score ≥ baseline; all refusals refuse.
- **Source-match:** recall@k — expected chunk retrieved per answerable question.
- **Isolation:** two-tenant seed; tenant A never retrieves tenant B.
- **Refusal/injection:** payload inside an uploaded doc → no leaked prompt, instruction ignored.

See [`../docs/08-testing-strategy.md`](../docs/08-testing-strategy.md).

---

## Possible future improvements

1. **Hybrid retrieval** (dense + BM25) for exact terms, codes, and names.
2. **Re-ranking** pass to lift answer precision on larger corpora.
3. **Groundedness scoring** to flag answers not entailed by cited chunks.
4. **Conversation memory/summarization** for long multi-turn sessions at bounded cost.
5. **Per-document access controls** within a workspace (not just per-workspace).
