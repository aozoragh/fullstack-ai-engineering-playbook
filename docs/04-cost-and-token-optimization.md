# Cost & Token Optimization

LLM cost scales with tokens and call volume, both of which are easy to let run unbounded. This checklist covers controlling spend **without** quietly degrading quality, and making cost visible per user and per feature so it can be managed like any other resource.

> Rule of thumb: every AI call should have a known, bounded worst-case token cost. If you can't state the worst case, it's unbounded.

---

## 1. Model selection

- [ ] Match the model to the task: use a smaller/cheaper model for classification, extraction, routing, and short answers; reserve the most capable model for hard reasoning and synthesis.
- [ ] Consider a **routing/tiering** approach — cheap model first, escalate to a stronger model only when needed.
- [ ] Pin model versions; "latest" can change cost and behavior under you.
- [ ] Re-evaluate model choice periodically — price/quality of available models shifts.

## 2. Input token control

- [ ] Only send what the task needs — trim boilerplate, system clutter, and redundant instructions.
- [ ] Bound retrieval context (`top_k` + per-chunk size) so input can't balloon with document size.
- [ ] Strip large, low-value content (raw HTML, navigation, repeated headers) before sending.
- [ ] Deduplicate near-identical chunks before they hit the prompt.

## 3. Output token control

- [ ] Set `max_output_tokens` on every call — never leave generation length unbounded.
- [ ] Ask for concise output / structured output when you don't need prose.
- [ ] For lists/extractions, request only the fields you'll use.

## 4. Context size

- [ ] Track the total context budget (system + history + retrieved + question + expected output) and keep it within a safe fraction of the model window.
- [ ] Reserve headroom for the output — don't fill the window with input.
- [ ] Larger context = higher cost and often *lower* quality (lost-in-the-middle). Bigger is not better by default.

## 5. Chat history truncation / summarization

- [ ] Conversation history is bounded — cap turns or token budget; don't resend the entire transcript every turn.
- [ ] Use rolling-window truncation for short sessions; **summarize** older turns for long sessions.
- [ ] Summaries are cached and updated incrementally, not regenerated from scratch each turn.
- [ ] Critical pinned context (system rules, key facts) is preserved even as history is trimmed.

## 6. Caching

- [ ] Cache embeddings for unchanged content (don't re-embed the same chunk).
- [ ] Cache responses for identical/normalized deterministic requests where correctness allows.
- [ ] Use provider **prompt caching** for large, stable prefixes (system prompt, shared context) to cut input cost.
- [ ] Cache keys include model + prompt version + inputs so cache invalidates correctly on change.

## 7. Batching embeddings

- [ ] Embedding generation is batched, not one call per chunk.
- [ ] Ingestion respects provider batch-size and rate limits to avoid throttling-driven retries.
- [ ] Bulk re-embedding (model migration) runs as a throttled background job with progress and a cost estimate.

## 8. Cost per user / per workflow

- [ ] Token usage and cost are **attributed** per user, workspace, and feature/workflow at call time.
- [ ] You can answer "what does the average user / this workflow cost per run?" from real data.
- [ ] Cost-per-action informs pricing and quota decisions — not guessed after the fact.
- [ ] Expensive workflows are profiled so the dominant cost driver is known.

## 9. Usage limits

- [ ] Per-user/workspace token and request quotas are enforced (also a security control — see security checklist).
- [ ] Global spend caps / circuit breakers stop runaway cost from bugs or abuse.
- [ ] Approaching-limit warnings exist before hard cutoffs.
- [ ] Limits fail safe and clear (`429` / quota message), never by silently dropping isolation or checks.

## 10. Latency monitoring

- [ ] Latency is tracked per model/endpoint (p50/p95/p99), alongside token counts.
- [ ] Slow calls are surfaced — latency and cost usually move together (bigger context, bigger output).
- [ ] Timeouts are set so slow provider calls don't pile up and amplify cost via retries.
- [ ] Streaming is used for long generations to improve perceived latency without changing cost.

## 11. Retry cost risks

- [ ] Retries are **bounded** (max attempts) with exponential backoff + jitter — no infinite loops.
- [ ] Retries only happen on retryable errors (timeouts, `429`, `5xx`), not on validation/logic failures.
- [ ] A retry storm cannot multiply cost — circuit breakers trip after repeated failures.
- [ ] Idempotency keys prevent a retried expensive action from being charged/executed twice (see API patterns).
- [ ] Workflow/agent loops have a hard step/iteration cap so they cannot recurse into unbounded spend.

---

## Where cost actually goes (typical RAG/chat app)

| Driver | Why it grows | Control |
| --- | --- | --- |
| Retrieved context | Large `top_k`, big chunks | Bound `top_k`, threshold, chunk size |
| Chat history | Full transcript resent each turn | Truncate / summarize |
| Output length | Unbounded generation | `max_output_tokens`, concise prompts |
| Re-embedding | Re-processing unchanged docs | Cache + idempotent ingestion |
| Retry storms | Retrying non-retryable errors | Bounded retries + circuit breaker |
| Wrong model | Big model for trivial tasks | Tier / route by task |

## Decision rules

- **Every call has a bounded worst-case token cost.** If not, fix that first.
- **Smallest model that passes eval wins.** Don't pay for capability you don't use.
- **Bound context and output before optimizing anything else** — they dominate cost.
- **Make cost visible per user/feature** — you can't manage what you don't measure.
- **Retries are capped and idempotent**, or they become a cost incident.

See also: [`05-ai-evaluation-checklist.md`](05-ai-evaluation-checklist.md) (don't trade away quality), [`02-llm-app-security-checklist.md`](02-llm-app-security-checklist.md) (quotas/abuse), and [`07-api-design-patterns.md`](07-api-design-patterns.md) (idempotency, usage tracking).
