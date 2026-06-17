# AI Evaluation Checklist

You cannot improve — or safely change — what you don't measure. AI evaluation is how you turn "it seems better" into a release decision. This checklist favors **cheap, deterministic checks first**, human review where it matters, and LLM-as-judge only once it's validated against humans.

> Evaluation exists to answer one question before you ship: does this change beat the current production version without regressing safety, at acceptable cost? A high score that can't answer that is not the goal — the **gate** is.

---

## Evaluation layers (cheapest → most expensive)

### 1. Deterministic checks
Fast, free, run on every change.

- [ ] Output **parses** and validates against the expected schema (valid JSON, required fields, types).
- [ ] Format/constraint checks: length bounds, allowed values, no forbidden tokens, no leaked system prompt.
- [ ] Refusal triggers on inputs that *should* be refused (out-of-scope, empty context, unsafe request).
- [ ] Latency and token counts are within budget.

### 2. Reference-based checks
Compare output to known-good answers on a fixed set.

- [ ] Exact/normalized match for closed-form answers (numbers, classifications, extractions).
- [ ] Similarity (embedding/string) for free-form answers, with a defensible threshold.
- [ ] Field-level accuracy for structured extraction (precision/recall per field).
- [ ] Results are reported **vs. the current production version**, not just absolute.

### 3. Retrieval / source checks (RAG)
Most "bad answers" are bad retrieval — measure it separately.

- [ ] **Recall@k / source-match**: did retrieval surface the chunk(s) that contain the answer?
- [ ] Citation correctness: do the cited sources actually support the answer?
- [ ] Groundedness/faithfulness: is the answer entailed by the retrieved context (not invented)?
- [ ] Refusal correctness: empty/low-confidence retrieval leads to refusal, not fabrication.

### 4. Human review
For quality dimensions automation can't fully judge.

- [ ] A rubric exists (correctness, completeness, tone, safety) — reviewers score consistently.
- [ ] A sampled set is human-reviewed each release; disagreements calibrate the rubric.
- [ ] Human labels become the ground truth that validates any automated judge.

### 5. LLM-as-judge (only when validated)
Useful for scale, dangerous when trusted blindly.

- [ ] The judge is **validated against human labels** and its agreement rate is known before it gates anything.
- [ ] The judge uses a fixed rubric and a pinned model/prompt version (it's product logic too — version it).
- [ ] The judge is checked for bias (position, verbosity, self-preference) on a control set.
- [ ] LLM-judge scores are treated as a signal, not gospel — borderline cases escalate to humans.

---

## Golden question sets

- [ ] A representative **golden set** exists per feature: real-world questions + expected answers (+ expected sources for RAG).
- [ ] It includes hard cases, edge cases, and **"should refuse"** cases — not just easy happy-path questions.
- [ ] It's versioned in the repo and grows when bugs/incidents reveal gaps (every production failure → a new golden case).
- [ ] It's large enough to be meaningful but small enough to run on every change.

## RAG evaluation

- [ ] Retrieval and generation are scored **separately** (recall@k vs. answer quality).
- [ ] Groundedness/faithfulness is measured, not assumed.
- [ ] Refusal behavior on out-of-scope questions is part of the set.
- [ ] Cross-tenant isolation is verified (no other tenant's content can be retrieved).

## Workflow / automation evaluation

- [ ] Each workflow has expected outcomes for representative inputs (correct action, correct extraction, correct routing).
- [ ] Failure handling is evaluated: bad input, provider error, partial completion, idempotent retry.
- [ ] Human-approval gates are exercised — sensitive actions don't fire without approval.
- [ ] End-to-end runs are checked for side effects (did it do the thing, exactly once?).

## Release gates for prompt / model / retrieval changes

Any change to a prompt, model, or retrieval config must pass the gate before promotion:

- [ ] Deterministic + reference checks pass.
- [ ] Golden-set score **≥ current production** (no regression), including refusal cases.
- [ ] Safety/guardrail cases still pass (injection, refusal, no leaked secrets).
- [ ] Cost and latency deltas are within budget (a quality win that doubles cost is a decision, not automatic).
- [ ] Results are attached to the PR; a human signs off on borderline trade-offs.
- [ ] Rollout is staged (canary → full) with the rollback path identified.

```
change → deterministic checks → reference + RAG/workflow eval → safety cases
       → cost/latency check → review → canary → full rollout
                              ↑ fail anywhere → block + rollback
```

---

## Decision rules

- **Measure retrieval separately from generation.** Fixing the wrong layer wastes time.
- **No promotion without beating the current production version on the golden set.**
- **LLM-as-judge must earn trust** by agreeing with humans before it gates anything.
- **Every production failure becomes a golden-set case** so it can't silently regress again.
- **A score is not a release decision** — the gate (beats baseline, safe, affordable) is.

See also: [`03-prompt-versioning-strategy.md`](03-prompt-versioning-strategy.md), [`01-rag-architecture-checklist.md`](01-rag-architecture-checklist.md), and [`08-testing-strategy.md`](08-testing-strategy.md).
