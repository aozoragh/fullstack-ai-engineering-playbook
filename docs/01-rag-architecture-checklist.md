# RAG Architecture Checklist

A practical, end-to-end checklist for building a Retrieval-Augmented Generation system that is **correct, isolated, and honest about what it doesn't know**. Use it during design, implementation, and review.

The property everything below protects: **the answer must be grounded in retrieved content, and the system must refuse when it can't ground it.**

---

## 1. Document ingestion

- [ ] Ingestion is an explicit, observable pipeline (queued job), not a side effect of an upload request.
- [ ] Each document has a stable ID, source, owner/workspace, version, and ingestion status.
- [ ] Re-ingestion is idempotent — re-uploading the same document doesn't create duplicate chunks.
- [ ] Failed ingestion is recorded with a reason and is retryable; partial failures don't leave half-indexed documents.
- [ ] Large files are processed in the background with progress/status surfaced to the user.

## 2. File type support

- [ ] The **supported file types are explicitly enumerated** (e.g. PDF, DOCX, TXT, MD, HTML) — not "everything."
- [ ] Unsupported types are rejected with a clear message, not silently dropped or partially parsed.
- [ ] File size and page-count limits are enforced (see cost and security checklists).
- [ ] Scanned/image PDFs are detected; OCR is either supported explicitly or the file is flagged as not extractable.

## 3. Text extraction

- [ ] Extraction quality is verified, not assumed — spot-check that tables, headers, and multi-column layouts come out readable.
- [ ] Extraction preserves enough structure (headings, page numbers) to support citations.
- [ ] Empty or near-empty extraction (e.g. image-only PDF) is detected and surfaced rather than indexed as blank.
- [ ] Encoding issues (smart quotes, ligatures, non-UTF-8) are normalized.

## 4. Chunking strategy

- [ ] Chunking is **semantic-aware** (by heading/paragraph/sentence) rather than naive fixed-byte splits where possible.
- [ ] Chunk size and overlap are tuned to the embedding model's context and the query type — documented, not arbitrary.
- [ ] Overlap is large enough to avoid cutting answers in half, small enough to avoid cost blowup.
- [ ] Each chunk stays self-contained enough to be meaningful when retrieved alone.
- [ ] Chunking is deterministic and versioned — changing the strategy triggers re-indexing, not silent drift.

## 5. Metadata preservation

- [ ] Every chunk carries: document ID, source title, page/section, position, workspace/tenant ID, and version.
- [ ] Metadata is stored alongside the vector so it can be used for filtering **and** citation.
- [ ] Timestamps allow "as of" reasoning and staleness checks.

## 6. Embeddings

- [ ] Embedding model is pinned and versioned; the version is stored with each vector.
- [ ] Query and document embeddings use the **same model and normalization**.
- [ ] Embeddings are generated in batches to control cost and rate limits.
- [ ] Re-embedding on model change is a planned migration, not an in-place surprise.

## 7. Vector storage

- [ ] Vectors are stored with tenant/workspace ID as a first-class, indexed field.
- [ ] The index type and distance metric match the embedding model (e.g. cosine vs. dot product).
- [ ] Upserts are keyed so re-ingestion replaces rather than duplicates.
- [ ] Deleting a document removes its vectors (and respects data-deletion/GDPR requests).

## 8. Retrieval quality

- [ ] `top_k` is tuned and documented; retrieval returns scores, not just chunks.
- [ ] A **relevance/score threshold** exists — low-scoring matches are dropped, not passed through.
- [ ] Retrieved chunks are de-duplicated and, where useful, re-ranked.
- [ ] Retrieval is filtered by tenant/workspace **before** ranking, never after.
- [ ] Total retrieved context fits the model budget with room for the question and answer.

## 9. Hybrid search

- [ ] Consider combining dense (vector) with sparse/keyword (BM25) retrieval for queries with exact terms, codes, or names.
- [ ] Fusion strategy (e.g. reciprocal rank fusion) is defined when both are used.
- [ ] The decision to use hybrid vs. pure vector is justified by query patterns, not added by default.

## 10. Source-aware answers

- [ ] The model is instructed to answer **only** from the provided context.
- [ ] Every answer can return its supporting sources (document + section/page).
- [ ] Citations are verifiable — they point to the actual chunk used, not a guess.
- [ ] The UI exposes sources so users can check the answer.

## 11. Weak-context refusal

- [ ] When retrieval is empty or below threshold, the system **refuses or asks for clarification** instead of answering from model priors.
- [ ] The refusal message is helpful ("I couldn't find this in your documents") rather than a generic error.
- [ ] Refusal behavior is covered by tests (see testing strategy).

## 12. Hallucination reduction

- [ ] Prompt explicitly forbids inventing facts not in context.
- [ ] Answers prefer "not found" over plausible fabrication.
- [ ] Optional: a grounding/faithfulness check verifies the answer is supported by the cited chunks.
- [ ] Numbers, names, dates, and quotes in answers are traceable to source chunks.

## 13. Workspace / user isolation

- [ ] Retrieval **cannot** return another tenant's chunks under any query — enforced in the data layer, not the prompt.
- [ ] Isolation is verified by a test that seeds two tenants and confirms no cross-leak.
- [ ] Shared/global knowledge (if any) is an explicit, separate, opt-in source — never the default mix.

## 14. Evaluation set

- [ ] A **golden question set** exists: representative questions with expected answers and expected source documents.
- [ ] It includes "should refuse" questions (out-of-scope, no supporting docs).
- [ ] Retrieval is measured (recall@k / source-match) separately from answer quality.
- [ ] The eval runs before any change to chunking, embeddings, retrieval, or the prompt ships.

## 15. Production failure modes (design for these)

| Failure | Symptom | Mitigation |
| --- | --- | --- |
| Empty retrieval | Confident but wrong answer | Threshold + refusal path |
| Stale index | Answers from deleted/old docs | Re-index on update; delete vectors on doc delete |
| Cross-tenant leak | User sees another tenant's data | Tenant filter in data layer + isolation test |
| Prompt injection via document | Model follows instructions inside a doc | Treat retrieved text as untrusted; guardrails |
| Chunk too small/large | Half-answers or bloated cost | Tuned chunk size + overlap |
| Embedding/model drift | Silent quality drop after model change | Pin + version embeddings; re-embed as migration |
| Cost blowup | High bill from large `top_k` / context | Bounded `top_k`, context budget, monitoring |
| Provider timeout | Hanging requests | Timeout + retry + fallback message |

---

## Decision rules

- **If retrieval confidence is low, refuse — don't guess.** A wrong grounded-sounding answer is worse than "I don't know."
- **Isolate in the data layer, not the prompt.** Prompts are not a security boundary.
- **Re-index on any change to chunking or embeddings.** Mixed strategies in one index produce inconsistent retrieval.
- **Measure retrieval and generation separately.** Most "bad answers" are actually bad retrieval.

See also: [`02-llm-app-security-checklist.md`](02-llm-app-security-checklist.md), [`05-ai-evaluation-checklist.md`](05-ai-evaluation-checklist.md), and [`../examples/rag-chatbot-architecture.md`](../examples/rag-chatbot-architecture.md).
