# GitHub PR Review Workflow

How I review (and expect reviews of) changes to AI products. A good review is ordered by risk, specific in its findings, and especially careful about the things that don't show up in a passing test: prompt changes, retrieval changes, and cost.

> A PR isn't "done" because it works on the author's machine. It's done when a reviewer can see *why* it's correct, *how* it's bounded, and *how* to undo it.

---

## Review priorities (in order)

Review top-down. Don't spend the first ten comments on style while a tenant-isolation bug sits unread.

1. **Correctness** — does it do what the PR says, including edge and failure cases?
2. **Security** — secrets server-side, untrusted input handled, output safe.
3. **Data isolation** — every query tenant-scoped; no cross-tenant path.
4. **API design** — consistent, idempotent where needed, structured errors, paginated.
5. **AI-specific behavior** — prompt/RAG/cost changes (below).
6. **Testing** — happy path *and* refusal/failure paths covered.
7. **Deployment readiness** — config, migrations, observability, rollback.
8. **Maintainability** — clarity, consistency, justified complexity.

### Correctness
- Trace the full path: input → validation → AI call → output → persistence.
- Check edge cases: empty input, huge input, concurrent requests, partial failure.
- Confirm the change matches the spec/intent, not just compiles.

### Security
- No secret/key reachable from client code or committed.
- Untrusted input (user + retrieved docs) handled as untrusted; output encoded before render.
- New endpoints rate-limited; expensive actions bounded.

### Data isolation
- Every new read/write filtered by tenant in the data layer.
- Retrieval filtered by tenant **before** ranking.
- Isolation has a test, or the PR adds one.

### API design
- Status codes and error shapes consistent with the rest of the API.
- Expensive/irreversible actions accept an idempotency key.
- List endpoints paginated and bounded.

---

## AI-specific review checklist

### Prompt changes
- [ ] Prompt is referenced by **ID + version**, not inlined as a raw string.
- [ ] **Eval results attached** — golden-set score vs. current production (no regression), including refusal cases.
- [ ] Model + settings (model id, temperature, max tokens) changes called out explicitly.
- [ ] Output schema unchanged, or bumped with consumers updated.
- [ ] No weakened guardrail/refusal/injection protection.
- [ ] Rollback path identified (previous version reversible).
> A prompt change with no eval results does not merge.

### RAG changes
- [ ] Chunking/embedding changes trigger a **re-index plan** (no mixed strategies in one index).
- [ ] Retrieval still tenant-filtered before ranking; threshold + `top_k` sane.
- [ ] Source-match / recall@k checked on the golden set; groundedness not regressed.
- [ ] Weak-context refusal still fires.

### Cost risks
- [ ] No unbounded fan-out (per-row LLM calls, recursive workflows, retry storms).
- [ ] Context and output are bounded (`max_output_tokens`, `top_k`, history truncation).
- [ ] Token/cost delta estimated; large increases justified.
- [ ] Idempotency protects expensive actions from double execution on retry.

### Test plan (required in the PR)
- [ ] What was tested and how (unit/integration/E2E).
- [ ] Refusal and failure paths covered, not just happy path.
- [ ] For AI changes: eval/golden-set results attached.
- [ ] How a reviewer can reproduce/verify locally.

### Rollback plan (required for risky changes)
- [ ] How to revert: code, **and** prompt/model/retrieval config independently.
- [ ] Migration down-path or forward-fix described.
- [ ] Rollout staging (canary → full) and rollback triggers stated.

---

## Reviewer etiquette

- Label findings: **blocking** / **should-fix** / **nit**. Don't block on nits.
- Be specific: cite `file:line`, the failure case, and a concrete fix.
- Distinguish "this is wrong" from "I'd prefer." Preferences are suggestions.
- Approve when blocking items are resolved — don't gatekeep on taste.

---

## Example PR template

Copy into `.github/pull_request_template.md`:

```markdown
## What & why
<!-- The change and the problem it solves. Link the issue/spec. -->

## Type of change
- [ ] Feature  - [ ] Fix  - [ ] Refactor
- [ ] Prompt change  - [ ] Model change  - [ ] Retrieval/RAG change
- [ ] Schema/migration

## How it works
<!-- Brief design notes. Trace the request path if non-trivial. -->

## Security & isolation
- [ ] No secrets in client/committed code
- [ ] New reads/writes are tenant-scoped (data layer)
- [ ] Untrusted input (user + retrieved docs) handled; output safe

## AI-specific (if applicable)
- [ ] Prompt referenced by id+version; model/settings changes noted
- [ ] **Eval results attached** (golden-set vs. production baseline, incl. refusal cases)
- [ ] RAG: re-index plan; retrieval tenant-filtered; source-match checked
- [ ] Cost: context/output bounded; no unbounded fan-out; idempotent expensive actions

## Test plan
<!-- What you tested, incl. refusal/failure paths. How to reproduce. -->
- [ ] Unit  - [ ] Integration  - [ ] E2E  - [ ] Eval/golden set

## Deployment & rollback
- [ ] Config in `.env.example`; migrations reversible; indexes added
- [ ] Observability for the new path (logs/metrics/cost)
- [ ] Rollback plan: <how to revert code + prompt/model/retrieval>

## Claims check
- [ ] No overstated claims ("handles all files", "prevents hallucination") — backed by code/tests/eval
```

---

## Decision rules

- **Order review by blast radius**: isolation/security/correctness before style.
- **No prompt/RAG change merges without eval results.**
- **Every risky change has a rollback plan in the PR.**
- **Specific, labeled findings** — blocking vs. nit, with `file:line` and a fix.

See also: [`../templates/pr-review-checklist.md`](../templates/pr-review-checklist.md), [`03-prompt-versioning-strategy.md`](03-prompt-versioning-strategy.md), and [`../AGENTS.md`](../AGENTS.md).
