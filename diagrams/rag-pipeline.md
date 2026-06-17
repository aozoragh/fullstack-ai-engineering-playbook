# RAG Pipeline

End-to-end flow for a tenant-isolated RAG system: ingestion on the left, query/answer on the right, with the **weak-context refusal** path made explicit. Renders on GitHub via Mermaid.

See [`../docs/01-rag-architecture-checklist.md`](../docs/01-rag-architecture-checklist.md) and [`../examples/rag-chatbot-architecture.md`](../examples/rag-chatbot-architecture.md).

```mermaid
flowchart TD
    subgraph Ingestion["Ingestion (async, per tenant)"]
        U[Upload file] --> V{"Valid type & size?"}
        V -- No --> RJ[Reject with clear error]
        V -- Yes --> EX[Extract text]
        EX --> EXC{Text extracted?}
        EXC -- No / empty --> FLAG["Flag: not extractable / OCR"]
        EXC -- Yes --> CH["Chunk + attach metadata<br/>doc id, page, tenant id, version"]
        CH --> EM[Embed in batches]
        EM --> UP[("Vector store<br/>upsert, idempotent")]
        CH --> DB[("Primary DB<br/>doc + chunk metadata")]
    end

    subgraph Query["Query (sync, streamed)"]
        Q[User question] --> QE[Embed query]
        QE --> SR["Vector search<br/>FILTER by tenant FIRST"]
        SR --> TH{"Score &ge; threshold?"}
        TH -- No / empty --> REF["Refuse:<br/>Not found in your documents"]
        TH -- Yes --> CTX["Assemble context<br/>retrieved = untrusted reference"]
        CTX --> LLM["LLM call<br/>answer only from context<br/>bounded output"]
        LLM --> SCHEMA{"Output valid<br/>vs schema?"}
        SCHEMA -- No --> ERR[Reject + safe error]
        SCHEMA -- Yes --> ANS[Answer + sources]
        ANS --> LOG["Log usage:<br/>tokens, latency, prompt ver, refused?"]
        REF --> LOG
    end

    UP -. retrieved by .-> SR

    classDef guard fill:#fde8e8,stroke:#c0392b,color:#7b241c;
    classDef ok fill:#e8f8f0,stroke:#1e8449,color:#145a32;
    class V,EXC,TH,SCHEMA guard;
    class ANS,REF ok;
```

## What the diagram encodes

- **Tenant filter happens before ranking**, in the data layer — not in the prompt.
- **Two decision gates protect quality:** the score threshold (→ refuse) and schema validation (→ safe error).
- **Refusal is a first-class path**, not an error — and it's logged like any answer.
- **Ingestion is idempotent** (upsert), so re-uploads don't duplicate chunks.
- **Retrieved content is treated as untrusted reference**, mitigating prompt injection via documents.
