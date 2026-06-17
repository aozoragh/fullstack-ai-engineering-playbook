# PR Review Checklist

> Paste into a PR review comment, or keep open while reviewing. Work top-down — risk before style. Label each finding **blocking** / **should-fix** / **nit**. See [`docs/09-github-pr-review-workflow.md`](../docs/09-github-pr-review-workflow.md).

## Correctness
- [ ] Does what the PR/spec says, including edge cases (empty, huge, concurrent, partial failure).
- [ ] Request path traced: input → validation → AI call → output → persistence.
- [ ] No obvious logic errors, off-by-ones, or unhandled branches.

## Security
- [ ] No API key/secret in client code, logs, or committed files.
- [ ] User input **and** retrieved content treated as untrusted.
- [ ] Model output encoded/validated before render or use (no XSS, no eval/exec).
- [ ] New endpoints rate-limited; expensive actions bounded.

## Data isolation (multi-tenant)
- [ ] Every new read/write filtered by tenant **in the data layer**.
- [ ] Retrieval filtered by tenant before ranking.
- [ ] Object-level authorization (no IDOR); IDs from other tenants 404.
- [ ] Isolation test present or added.

## API design
- [ ] Status codes + error shape consistent with the rest of the API.
- [ ] Expensive/irreversible actions accept an idempotency key.
- [ ] List endpoints paginated and bounded.
- [ ] Long-running work is an async job, not a blocking request.

## AI-specific
**Prompt changes**
- [ ] Referenced by id + version (not inlined).
- [ ] Eval results attached (golden set vs. production baseline, incl. refusal cases).
- [ ] Model/settings/schema changes called out.
- [ ] No weakened guardrail/refusal.

**RAG changes**
- [ ] Chunking/embedding change has a re-index plan.
- [ ] Retrieval tenant-filtered; threshold/top_k sane.
- [ ] Source-match / groundedness not regressed; refusal still fires.

**Cost**
- [ ] No unbounded fan-out (per-row calls, recursion, retry storms).
- [ ] Context/output bounded; token/cost delta justified.
- [ ] Idempotency protects against double-charge on retry.

## Testing
- [ ] New paths have tests, including refusal/failure — not just happy path.
- [ ] Providers mocked (deterministic CI).
- [ ] Eval/golden-set results attached for AI changes.
- [ ] Test plan in the PR explains how to reproduce.

## Deployment readiness
- [ ] New config in `.env.example`, read server-side.
- [ ] Migrations reversible; required indexes added.
- [ ] Timeouts, bounded retries, fallback for provider failure.
- [ ] Observability for the new path (logs/metrics/cost).
- [ ] Rollback plan (code + prompt/model/retrieval).

## Maintainability
- [ ] Names/structure/error handling consistent with surrounding code.
- [ ] Complexity justified; no dead code.
- [ ] Prompt/model logic centralized, not scattered inline.

## Claims check
- [ ] No overstated claims ("handles all files", "prevents hallucination", "fully secure") unbacked by code/tests/eval.

---
**Summary:** blocking count = ___ · highest-risk item = ___ · safe to merge after blockers resolved? Y / N
