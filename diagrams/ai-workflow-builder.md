# AI Workflow Builder

Flow for an AI automation run: **trigger → classify → retrieve → draft → risk gate → act**, with a mandatory human-approval branch for sensitive actions and idempotent, exactly-once side effects. Renders on GitHub via Mermaid.

See [`../examples/ai-workflow-automation-architecture.md`](../examples/ai-workflow-automation-architecture.md) and [`../docs/02-llm-app-security-checklist.md`](../docs/02-llm-app-security-checklist.md).

```mermaid
flowchart TD
    T["Trigger<br/>ticket / event / schedule"] --> IDEM{"Idempotency key<br/>seen before?"}
    IDEM -- Yes --> CACHED["Return prior result<br/>no re-execution"]
    IDEM -- No --> RUN["Create run<br/>state: queued"]

    RUN --> CL["Classify<br/>intent / priority / RISK<br/>cheap model"]
    CL --> RT["Retrieve<br/>tenant-filtered KB + threshold"]
    RT --> DR["Draft action/reply<br/>grounded + citations<br/>schema-validated"]

    DR --> GATE{"Risk level + confidence"}
    GATE -- Low risk + high conf --> AUTO[Auto-approve]
    GATE -- Sensitive / low conf --> HUMAN["Human approval queue<br/>preview the action"]

    HUMAN --> DEC{Approved?}
    DEC -- No --> REJ["Reject<br/>log + no side effect"]
    DEC -- Yes --> EXEC
    AUTO --> EXEC["Action executor<br/>allowlisted - EXACTLY ONCE"]

    EXEC --> SE{Side effect ok?}
    SE -- Error --> SAFE["Safe state<br/>bounded retry / resume"]
    SE -- Ok --> DONE["state: done"]

    DONE --> AUD["Audit + usage log<br/>who/what/when - tokens - cost"]
    REJ --> AUD
    CACHED --> AUD

    LOOP["Step / iteration cap"] -. bounds .-> CL
    LOOP -. bounds .-> DR

    classDef guard fill:#fde8e8,stroke:#c0392b,color:#7b241c;
    classDef ok fill:#e8f8f0,stroke:#1e8449,color:#145a32;
    classDef human fill:#fef9e7,stroke:#b9770e,color:#7e5109;
    class IDEM,GATE,DEC,SE guard;
    class DONE,AUTO ok;
    class HUMAN human;
```

## What the diagram encodes

- **Idempotency at the entry** — a repeated trigger returns the prior result, never re-runs side effects.
- **Risk classification drives the approval gate** — sensitive/low-confidence actions cannot auto-execute.
- **Human approval previews the exact action** before it fires; rejection produces no side effect.
- **The executor is allowlisted and exactly-once**, so retries can't double-send.
- **A step/iteration cap bounds the run**, preventing runaway loops and cost.
- **Everything ends in an audit + usage log** — auto, approved, rejected, or cached.
