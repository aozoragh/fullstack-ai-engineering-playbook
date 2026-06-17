# Prompt Versioning Strategy

Prompts are **product logic**. A prompt change can alter behavior as much as a code change — sometimes more — yet it is often edited inline, untracked, and shipped without review or evaluation. This document describes treating prompts as versioned, reviewable, testable artifacts.

---

## Why prompts are product logic

- A prompt edit can change correctness, tone, refusal behavior, output format, and cost — for every user, instantly.
- Bugs in prompts are real bugs: broken JSON, leaked instructions, regressions in accuracy.
- Without versioning you cannot answer the three questions that matter after an incident: **What changed? When? Can we roll it back?**

Treat every production prompt the way you treat a deployed function: identified, versioned, reviewed, evaluated, and reversible.

---

## Prompt IDs and versions

- [ ] Each prompt has a stable **ID** (e.g. `rag.answer`, `support.triage`) that code references — code never contains the raw prompt string scattered inline.
- [ ] Each prompt has a **version** (semantic or incrementing) recorded with every change.
- [ ] Prompts live in a single source of truth (a versioned file/registry/store), not copy-pasted across services.
- [ ] The prompt ID + version is **logged with every model call**, so any output can be traced to the exact prompt that produced it.

## Model settings travel with the prompt

A prompt is only reproducible together with its runtime settings. Version them as one unit:

- [ ] Model name **and** version/snapshot (not just "the latest").
- [ ] Temperature, top-p, max tokens, stop sequences.
- [ ] Tool definitions / function schemas available to the call.
- [ ] Any retrieval parameters that shape the context (`top_k`, threshold) if they're part of this behavior.

## Output schema tracking

- [ ] If the prompt returns structured output, its **schema is versioned alongside the prompt**.
- [ ] Output is validated against the schema at runtime; schema violations are logged and handled.
- [ ] Schema changes are breaking changes — bumped, reviewed, and coordinated with consumers.

## Rollback strategy

- [ ] The previous prompt version is always retrievable and re-deployable.
- [ ] Rolling back a prompt is a config/flag change, not a code redeploy where possible.
- [ ] Each prompt version is tied to its evaluation results, so you know the last known-good version.
- [ ] Rollback is part of the deploy plan for any prompt change (see deployment readiness).

## Evaluation before promotion

- [ ] No prompt version reaches production without passing the relevant **golden set** (see evaluation checklist).
- [ ] Eval compares the new version against the current production version (regression check), not just an absolute bar.
- [ ] Both quality **and** cost/latency are checked — a "better" prompt that doubles tokens is a trade-off, not a free win.
- [ ] Promotion is gated: eval pass → review → staged rollout → full.

## Prompt change review checklist

Use this in the PR for any prompt change:

- [ ] **What changed and why** — the behavioral intent, not just the diff.
- [ ] **Eval results** — golden-set pass/fail vs. current production, attached.
- [ ] **Cost/latency delta** — token count change estimated or measured.
- [ ] **Output schema** — unchanged, or bumped + consumers updated.
- [ ] **Injection/safety** — does the change weaken any guardrail or refusal behavior?
- [ ] **Model settings** — any change to model/temperature/limits called out explicitly.
- [ ] **Rollback** — previous version identified and reversible.

A prompt PR with no eval results should not merge.

---

## Example prompt metadata block

Store prompts with metadata so every call is reproducible and traceable. Example (YAML; adapt to your stack):

```yaml
id: rag.answer
version: 4
description: >
  Answers a user question strictly from retrieved context.
  Refuses when context is empty or low-confidence.
owner: ai-platform
model:
  name: claude-opus-4-8
  temperature: 0.2
  max_output_tokens: 800
output_schema: schemas/rag_answer.v2.json
inputs:
  - question        # user question (untrusted)
  - context_chunks  # retrieved, tenant-filtered (untrusted)
guardrails:
  - answer_only_from_context
  - refuse_on_weak_context
  - never_reveal_system_prompt
eval:
  golden_set: evals/rag_answer/golden.jsonl
  last_run: "2026-06-10"
  pass_rate: 0.94
  baseline_version: 3
changelog:
  - v4: tightened refusal threshold; added citation requirement
  - v3: switched to structured output with sources[]
```

And the runtime log line that ties output to prompt:

```json
{
  "request_id": "req_8f...",
  "prompt_id": "rag.answer",
  "prompt_version": 4,
  "model": "claude-opus-4-8",
  "input_tokens": 1840,
  "output_tokens": 210,
  "latency_ms": 1320,
  "outcome": "answered",
  "workspace_id": "ws_123"
}
```

---

## Decision rules

- **No raw prompt strings inline.** Reference an ID + version.
- **Prompt + model settings + schema are one versioned unit.**
- **No promotion without an eval that beats the current production version.**
- **Every output is traceable to the exact prompt version that produced it.**

See also: [`05-ai-evaluation-checklist.md`](05-ai-evaluation-checklist.md) and [`09-github-pr-review-workflow.md`](09-github-pr-review-workflow.md).
