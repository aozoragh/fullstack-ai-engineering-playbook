# LLM Application Security Checklist

Security for AI products covers the usual web-app concerns **plus** a new attack surface: the model itself, the untrusted content you feed it, and the actions it can take. This checklist focuses on the AI-specific risks layered on top of standard application security.

> Assume the prompt is **not** a security boundary. Anything you rely on the model to "not do" can be overridden by crafted input.

---

## 1. API keys & provider secrets stay server-side

- [ ] LLM/vector/provider keys exist **only** on the server (env vars / secret manager), never in client bundles, mobile apps, or public repos.
- [ ] The browser calls *your* backend; your backend calls the provider. No direct client → provider calls with a shared key.
- [ ] Secrets are not committed: `.env` is gitignored, history is scanned, and a secret scanner runs in CI.
- [ ] Keys are scoped and rotatable; usage is attributable so a leaked key can be revoked without taking everything down.

## 2. PII & sensitive data handling

- [ ] Only the data the task actually needs is sent to the model — strip or redact unnecessary PII before the call.
- [ ] Know your provider's data-retention/training policy; use zero-retention / no-training options for sensitive data where available.
- [ ] Sensitive fields (SSNs, payment data, health data, credentials) are never embedded into a vector store unless explicitly required and access-controlled.
- [ ] Data deletion requests propagate to vectors, caches, and logs — not just the primary DB.

## 3. Logging rules

- [ ] **Do not log full prompts/responses containing user PII by default.** If you must, redact and access-restrict.
- [ ] Never log secrets, raw auth tokens, or full request bodies with sensitive fields.
- [ ] Logs capture enough to debug (request ID, model, token counts, latency, outcome) without capturing the sensitive payload.
- [ ] Log access is restricted and retention is bounded.

## 4. Prompt injection risks

- [ ] Assume **any** input can contain instructions ("ignore previous instructions," "reveal your system prompt," "email this to…").
- [ ] Keep a clear, structured separation between system instructions and user/retrieved content; don't blend untrusted text into the instruction region.
- [ ] Don't expose the system prompt or secrets to the model output path where they can be echoed back.
- [ ] For tool-using agents, injected instructions must not be able to trigger sensitive tools without independent validation.
- [ ] Test known injection payloads as part of the suite (see testing strategy).

## 5. Retrieved documents are untrusted input

- [ ] Treat **all** retrieved chunks, web pages, emails, and tool outputs as attacker-controlled — they routinely contain injection payloads.
- [ ] Retrieved content cannot, on its own, escalate privileges, call tools, or change the system's instructions.
- [ ] Where feasible, label/segregate retrieved content so the model knows it's reference material, not commands.
- [ ] Outbound actions triggered by content (links, tool calls) are validated against an allowlist.

## 6. Rate limiting

- [ ] Per-user, per-IP, and per-endpoint rate limits exist on AI endpoints (they are expensive and abusable).
- [ ] Limits are enforced server-side and return clear `429` responses with retry guidance.
- [ ] Burst protection covers expensive operations (ingestion, embeddings, long generations) specifically.

## 7. Usage limits & quotas

- [ ] Per-user / per-workspace **usage quotas** (requests, tokens, documents) are enforced and tied to plan/entitlement.
- [ ] Hitting a quota fails safely with a clear message — it does not silently degrade isolation or skip checks.
- [ ] Global circuit breakers / spend caps protect against runaway cost from abuse or bugs.

## 8. File upload limits

- [ ] Max file size, page count, and per-user upload volume are enforced **before** processing.
- [ ] File type is validated by content, not just extension; dangerous types are rejected.
- [ ] Uploads are scanned/sandboxed; parsing happens in an isolated worker, not the request thread.
- [ ] Storage paths are namespaced per tenant; one user cannot read another's uploads.

## 9. Workspace isolation

- [ ] Every AI operation is scoped to the caller's workspace/tenant, enforced in the **data layer**.
- [ ] Retrieval, history, usage, and uploads are all tenant-filtered; none rely on the model or UI to stay separate.
- [ ] Cross-tenant isolation is verified by an automated test that seeds two tenants.
- [ ] Admin/impersonation paths are audited and logged.

## 10. Output safety

- [ ] Model output is treated as untrusted before rendering: encode/escape to prevent XSS; never `eval` or execute it.
- [ ] If output drives actions (SQL, shell, API calls, code), it is validated/parameterized — never passed through raw.
- [ ] Structured outputs are schema-validated before use; malformed output is rejected, not best-guessed.
- [ ] Sensitive content filters / refusals are applied where the product requires them.

## 11. Human approval for sensitive workflows

- [ ] Irreversible or high-impact actions (sending emails, payments, deletions, external posts) require **explicit human approval** before execution.
- [ ] The approval step shows exactly what will happen (preview the action + inputs).
- [ ] Approvals are logged with who/when for audit.
- [ ] Autonomous agents have a bounded action allowlist and cannot self-expand it.

---

## Quick threat model (fill in per feature)

| Question | Notes |
| --- | --- |
| What untrusted input enters the model? | User text, uploaded docs, retrieved chunks, tool output |
| What can the model *cause*? | Reads, writes, external calls, spend |
| What's the worst case if injected? | Data exfil, cross-tenant leak, unauthorized action, cost |
| Where's the boundary enforced? | Data layer / allowlist / human approval — **not the prompt** |

## Decision rules

- **Secrets never reach the client. Ever.**
- **Untrusted in → validated out.** Both the model's input and its output cross trust boundaries.
- **The prompt is guidance, not a guard.** Enforce security in code.
- **Irreversible actions need a human or a hard guardrail.**

See also: [`01-rag-architecture-checklist.md`](01-rag-architecture-checklist.md) (isolation), [`07-api-design-patterns.md`](07-api-design-patterns.md) (access control), and [`04-cost-and-token-optimization.md`](04-cost-and-token-optimization.md) (abuse/cost).
