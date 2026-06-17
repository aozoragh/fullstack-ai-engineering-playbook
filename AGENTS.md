# AGENTS.md — Review Instructions for AI Coding & Review Agents

This file tells AI coding/review agents (Claude, Codex, and similar) how to review code in projects that follow this playbook. The goal is a review that catches the failures that actually break AI products in production — not just style nits.

If you are an AI agent reviewing a change here or in a linked project, follow these instructions.

---

## How to review

1. **Read the diff and the surrounding context.** Do not review lines in isolation. Trace the request path: input → validation → retrieval/AI call → output → persistence.
2. **Prioritize by blast radius.** Security, data isolation, and correctness first. Style last.
3. **Be specific.** Cite `file:line`, state the concrete failure case, and propose a fix. "Looks fine" is not a review.
4. **Flag, don't fabricate.** If you are unsure whether something is a bug, say so explicitly and explain what would confirm it. Do not invent vulnerabilities to look thorough.
5. **Separate blocking issues from suggestions.** Label each finding `blocking`, `should-fix`, or `nit`.

---

## What to focus on

### 1. Production readiness
- Are errors handled, or will an unhandled exception reach the user / crash the worker?
- Are external calls (LLM, vector store, DB, third-party APIs) wrapped with timeouts and bounded retries?
- Is there a fallback when the AI provider is slow, rate-limited, or down?
- Are long-running AI actions run as async jobs rather than blocking a request?

### 2. Security & privacy
- **API keys and provider secrets must stay server-side.** Flag any secret, key, or token reachable from client code or committed to the repo.
- Is sensitive data (PII, credentials, internal IDs) logged, sent to the model unnecessarily, or returned in errors?
- Are inputs validated and outputs encoded/escaped before rendering?

### 3. Data isolation (multi-tenant)
- Every read/write must be scoped to the current workspace/tenant/user. Flag any query that can cross tenant boundaries.
- Vector/retrieval queries must filter by tenant **before** ranking, not after.
- Confirm access-control checks happen server-side, not just in the UI.

### 4. Prompt injection & untrusted input
- **Treat all retrieved documents, tool outputs, and user content as untrusted.** They can contain instructions.
- Flag prompts that concatenate untrusted content into the system/instruction region.
- Check that retrieved content cannot trigger tool calls, data exfiltration, or instruction override without guardrails.
- For agents with tools: are destructive/irreversible actions gated behind validation or human approval?

### 5. RAG correctness
- Is retrieval filtered by tenant and relevance before being passed to the model?
- Does the answer cite or ground in retrieved sources, or can it free-associate?
- Is there a **weak-context refusal path** — does the system decline when retrieval is empty or low-confidence instead of hallucinating?
- Are chunk boundaries and metadata preserved so citations point to the right place?

### 6. Cost & token control
- Are inputs/outputs bounded (max tokens, max context, history truncation)?
- Could this change cause unbounded fan-out (per-row LLM calls, retry storms, recursive workflows)?
- Is the most appropriate (often cheapest sufficient) model used for the task?
- Is usage attributable per user/workspace for cost accounting and limits?

### 7. API design
- Are resources, status codes, and error shapes consistent with the rest of the API?
- Are expensive/irreversible AI actions **idempotent** (idempotency key or dedupe)?
- Is pagination present on list endpoints? Are errors structured and machine-readable?

### 8. Testing
- Do new AI-facing paths have tests, including the **refusal** and **failure** cases — not just the happy path?
- For RAG changes: is there a source-matching / regression test on a golden set?
- Are external providers mocked so tests are deterministic?

### 9. Deployment readiness
- New config → is it documented in `.env.example` and read server-side?
- Schema change → is there a migration, and are required indexes added?
- Is there observability (logs/metrics/traces) for the new path, including token/cost?
- Is there a rollback plan for prompt/model/retrieval changes?

### 10. Maintainability
- Is prompt/model logic versioned and centralized, not hardcoded inline in random places?
- Are names, structure, and error handling consistent with the surrounding code?
- Is complexity justified, or can it be simplified?

### 11. Unsupported claims
- Flag comments, docs, or PR descriptions that **overstate** behavior: "handles all file types," "prevents hallucination," "fully secure," "100% accurate."
- Require claims to be backed by code, tests, or eval results. If a claim cannot be verified from the change, call it out.

---

## Output format

For each finding:

```
[blocking|should-fix|nit] <file>:<line> — <concise problem>
  Why it matters: <impact / failure case>
  Suggested fix: <concrete change>
```

End with a short summary: blocking issue count, the single highest-risk item, and whether the change is safe to merge after the blocking items are resolved.

---

## Do not

- Do not approve a change that loosens tenant isolation or exposes secrets, regardless of how small.
- Do not rewrite the whole file when a targeted fix will do.
- Do not pad the review with style nits while missing a security or isolation bug.
- Do not claim a change is "fully tested" or "secure" — describe what is and isn't covered.

---

## Reference

These instructions condense the playbook. For the full rationale and checklists behind each focus area:

| Focus area | Playbook doc |
| --- | --- |
| RAG correctness, data isolation | [`docs/01-rag-architecture-checklist.md`](docs/01-rag-architecture-checklist.md) |
| Security, privacy, prompt injection | [`docs/02-llm-app-security-checklist.md`](docs/02-llm-app-security-checklist.md) |
| Prompt changes | [`docs/03-prompt-versioning-strategy.md`](docs/03-prompt-versioning-strategy.md) |
| Cost & token control | [`docs/04-cost-and-token-optimization.md`](docs/04-cost-and-token-optimization.md) |
| Evaluation & release gates | [`docs/05-ai-evaluation-checklist.md`](docs/05-ai-evaluation-checklist.md) |
| Deployment readiness | [`docs/06-deployment-readiness-checklist.md`](docs/06-deployment-readiness-checklist.md) |
| API design | [`docs/07-api-design-patterns.md`](docs/07-api-design-patterns.md) |
| Testing | [`docs/08-testing-strategy.md`](docs/08-testing-strategy.md) |
| PR review workflow | [`docs/09-github-pr-review-workflow.md`](docs/09-github-pr-review-workflow.md) |
