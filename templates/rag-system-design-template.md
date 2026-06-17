# RAG System Design — <System Name>

> Copy into your repo and fill in. Every section should have a concrete decision and a one-line rationale, not "TBD." Pair with [`docs/01-rag-architecture-checklist.md`](../docs/01-rag-architecture-checklist.md).

| Field | Value |
| --- | --- |
| Owner / Reviewers | |
| Last updated | YYYY-MM-DD |
| Status | Draft / Approved |

---

## 1. Goal & scope
- **What questions must this answer well?**
- **What sources** feed it (doc types, volume, update frequency)?
- **Out of scope:**

## 2. Ingestion
| Decision | Choice | Rationale |
| --- | --- | --- |
| Supported file types (enumerated) | | |
| Extraction approach (incl. OCR?) | | |
| Trigger (upload / batch / scheduled) | | |
| Idempotency (re-ingest behavior) | | |
| Failure handling & retry | | |

## 3. Chunking
| Decision | Choice | Rationale |
| --- | --- | --- |
| Strategy (semantic / fixed / hybrid) | | |
| Chunk size & overlap | | |
| Versioned? (re-index on change) | | |
| Metadata per chunk | doc id, title, page/section, tenant id, version | |

## 4. Embeddings
| Decision | Choice | Rationale |
| --- | --- | --- |
| Model (pinned version) | | |
| Dimensions / normalization | | |
| Batching strategy | | |
| Re-embed plan on model change | | |

## 5. Vector store & indexing
| Decision | Choice | Rationale |
| --- | --- | --- |
| Store | | |
| Distance metric | | |
| Tenant field (indexed) | | |
| Upsert key (dedupe) | | |
| Delete propagation | | |

## 6. Retrieval
| Decision | Choice | Rationale |
| --- | --- | --- |
| `top_k` | | |
| Score threshold | | |
| Hybrid (dense + keyword)? | | |
| Re-ranking? | | |
| Tenant filter (before ranking) | yes — data layer | |
| Context budget (tokens) | | |

## 7. Generation
| Decision | Choice | Rationale |
| --- | --- | --- |
| Model & settings | | |
| Prompt id + version | | |
| Answer-only-from-context | yes | |
| Citation format (sources[]) | | |
| Weak-context refusal | yes — message: "…" | |
| Output schema | | |

## 8. Isolation & security
- Tenant isolation enforced at: (data-layer filter; verified by test)
- Retrieved content treated as untrusted (injection); cannot trigger actions.
- PII handling / redaction:
- Deletion / retention:

## 9. Evaluation
- **Golden set:** location, size, includes "should refuse" cases (Y/N)
- **Retrieval metric:** recall@k / source-match target =
- **Generation metric:** groundedness / answer accuracy target =
- **Release gate:** must beat baseline; refusal cases must pass.

## 10. Cost & performance
- Estimated tokens/cost per query:
- Worst-case context size:
- Latency target (p95):
- Caching (embeddings / responses / prompt prefix):

## 11. Failure modes
| Failure | Mitigation |
| --- | --- |
| Empty/low-confidence retrieval | refuse |
| Stale index after doc update/delete | re-index / delete vectors |
| Cross-tenant leak | data-layer filter + isolation test |
| Injection via document | untrusted handling + guardrails |
| Cost blowup | bounded top_k/context + monitoring |

## 12. Observability
- Per query: request id, tenant, top_k, scores, tokens, latency, refused?(bool)
- Alerts: error rate, latency, retrieval-empty rate, cost.

## 13. Rollout & rollback
- Re-index migration plan:
- Rollback for prompt/retrieval config:

---

### Sign-off
- [ ] Isolation verified by test
- [ ] Golden set + gate defined
- [ ] Cost bounded with worst-case known
- [ ] Re-index/rollback plan written
