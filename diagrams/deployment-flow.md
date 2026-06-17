# Deployment Flow

CI/CD flow for AI changes with an **evaluation gate**, staged rollout, and explicit rollback triggers. The key idea: a prompt/model/retrieval change must pass eval before it can reach production, and it must be reversible **independently of code**. Renders on GitHub via Mermaid.

See [`../docs/06-deployment-readiness-checklist.md`](../docs/06-deployment-readiness-checklist.md) and [`../docs/05-ai-evaluation-checklist.md`](../docs/05-ai-evaluation-checklist.md).

```mermaid
flowchart TD
    C[Commit / PR] --> CI["CI:<br/>lint - unit - integration<br/>providers mocked"]
    CI --> CIQ{Pass?}
    CIQ -- No --> FB1[Fail PR]
    CIQ -- Yes --> SEC["Secret scan +<br/>migration & index check"]
    SEC --> AI{"Prompt / model /<br/>retrieval changed?"}

    AI -- Yes --> EVAL["Eval gate:<br/>golden set vs baseline<br/>+ refusal + cost/latency"]
    AI -- No --> STAGE
    EVAL --> EVQ{"Beats baseline<br/>& safe & affordable?"}
    EVQ -- No --> FB2["Block:<br/>fix or roll back prompt"]
    EVQ -- Yes --> STAGE[Deploy to staging]

    STAGE --> SMOKE["Smoke tests<br/>ingest - ask - grounded - refuse"]
    SMOKE --> SMQ{Healthy?}
    SMQ -- No --> FB3[Block release]
    SMQ -- Yes --> CANARY["Canary / % traffic"]

    CANARY --> MON{"Triggers ok?<br/>error rate - p95 - cost - eval"}
    MON -- Breach --> RB["Rollback<br/>code AND prompt/model/retrieval<br/>independently"]
    MON -- Ok --> FULL[Full rollout]

    FULL --> OBS["Observe:<br/>logs - metrics - token/cost<br/>per tenant + feature"]
    RB --> OBS

    classDef guard fill:#fde8e8,stroke:#c0392b,color:#7b241c;
    classDef ok fill:#e8f8f0,stroke:#1e8449,color:#145a32;
    classDef gate fill:#eaf2f8,stroke:#2471a3,color:#1a5276;
    class CIQ,EVQ,SMQ,MON,AI guard;
    class FULL,OBS ok;
    class EVAL,CANARY gate;
```

## What the diagram encodes

- **The eval gate only triggers for AI changes** — code-only changes skip it, but prompt/model/retrieval changes cannot bypass it.
- **The gate checks three things:** beats baseline (quality), stays safe (refusal/guardrails), stays affordable (cost/latency).
- **Smoke tests run against a real environment** before any user traffic.
- **Canary precedes full rollout**, with explicit rollback triggers (error rate, p95 latency, cost, eval drop).
- **Rollback covers code *and* prompt/model/retrieval independently** — config-level revert, not only redeploy.
- **Observability is the endpoint of every path**, including rollback.
