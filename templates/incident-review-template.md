# Incident Review — <Short Title>

> Blameless post-incident review. Focus on systems and signals, not people. Copy into the incident ticket. The deliverable is the **action items** and the **regression test / golden case** that ensures this can't silently recur.

| Field | Value |
| --- | --- |
| Incident ID | |
| Severity | SEV1 / SEV2 / SEV3 |
| Status | Investigating / Mitigated / Resolved |
| Detected | YYYY-MM-DD HH:MM (UTC) — by: alert / user report / monitoring |
| Resolved | YYYY-MM-DD HH:MM (UTC) |
| Duration | |
| Authors | |

---

## 1. Summary
> 2–3 sentences: what happened, who/what was affected, how it was resolved.

-

## 2. Impact
- **Users/tenants affected:**
- **Scope:** (requests failed, wrong answers served, data exposed, cost overrun)
- **Data/privacy impact:** (any cross-tenant exposure, PII, leaked secret? — escalate if yes)
- **Cost impact:** (unexpected spend, if any)

## 3. Timeline (UTC)
| Time | Event |
| --- | --- |
| | Change deployed / trigger |
| | First symptom |
| | Detected |
| | Mitigation started |
| | Mitigated |
| | Resolved |

## 4. Root cause
> The actual cause, not the symptom. Use "5 whys" if useful. For AI incidents, name the layer.

- **Category:** code / config / prompt / model change / retrieval / data / provider / infra / cost / isolation
- **Root cause:**
- **Why it wasn't caught:** (gap in eval / test / monitoring / review)

## 5. Detection & response
- **How was it detected?** (and could it have been detected sooner?)
- **What made mitigation slow/fast?**
- **Was rollback available and used?** (code / prompt / model / retrieval)

## 6. Contributing factors
- (e.g. prompt change shipped without eval; no cost alert; missing isolation test; unbounded retry)

## 7. What went well
-

## 8. Action items
> Owned, dated, tracked. Prioritize prevention and detection.

| Action | Type (prevent/detect/mitigate) | Owner | Due | Ticket |
| --- | --- | --- | --- | --- |
| | | | | |

## 9. Regression guard (required)
- [ ] **Regression test added** (code-level) reproducing the failure.
- [ ] **Golden case added** (behavior-level) if AI/RAG/refusal-related.
- [ ] Alert/monitor added or tuned so this is detected automatically next time.

## 10. Lessons / playbook updates
- Checklist or doc to update:
- Pattern to add to review:

---
**Sign-off:** ____  ·  **Date:** ____

---

> **In this playbook:** pairs with the [deployment readiness](../docs/06-deployment-readiness-checklist.md) and [evaluation](../docs/05-ai-evaluation-checklist.md) checklists. Delete this line when you copy the template into a project.
