# Tessera — Architecture

## Service Map

```
                        ┌─────────────────────────────────────┐
                        │           tessera-web                │
                        │        (Next.js App Router)          │
                        │  Analyst dashboard + feedback UI     │
                        └────────────┬────────────┬────────────┘
                                     │            │
                          POST /analyze    POST /feedback
                                     │            │
                        ┌────────────▼────────────▼────────────┐
                        │          tessera-agent               │
                        │        (Python / FastAPI)            │
                        │                                      │
                        │  ┌──────────────────────────────┐   │
                        │  │        Agent Loop             │   │
                        │  │  system prompt + tools + RAG  │   │
                        │  └──────────────────────────────┘   │
                        │         │              │             │
                        │   Anthropic API    Langfuse SDK      │
                        └────────────┬─────────────────────────┘
                                     │
                          GET /cases, /user-history, etc.
                                     │
                        ┌────────────▼────────────────────────┐
                        │          tessera-data                │
                        │          (Go / net/http)             │
                        │                                      │
                        │  - Historical cases CRUD             │
                        │  - pgvector similarity search        │
                        │  - Feedback ingestion                │
                        └────────────┬────────────────────────┘
                                     │
                        ┌────────────▼────────────────────────┐
                        │       PostgreSQL + pgvector          │
                        │         (Docker Compose / dev)       │
                        └─────────────────────────────────────┘
```

## Data Flow — Analyze Request

1. Analyst (or upstream rule engine) sends transaction to `tessera-agent POST /analyze`
2. Agent builds context: calls tools on `tessera-data` (user history, IP fingerprint, blacklists)
3. Agent calls pgvector via `tessera-data` to retrieve top-k similar historical cases
4. Agent calls Claude with system prompt + tool results + retrieved cases
5. Claude returns a structured verdict → validated against Pydantic schema
6. If verdict has no cited source → escalate flag set, decision = `ESCALATE`
7. Langfuse trace written for every run
8. Verdict returned to caller

## Data Flow — Feedback Loop

1. Analyst reviews verdict in `tessera-web`
2. Analyst confirms/corrects decision + adds note + marks decisive signals
3. `tessera-web` sends corrected case to `tessera-agent POST /feedback`
4. `tessera-agent` forwards to `tessera-data POST /cases`
5. `tessera-data` embeds the case and inserts into pgvector store
6. Case is now retrievable for future similar transactions

## Service Boundaries

| Concern | Owner |
|---|---|
| Agent reasoning, prompt, tools | `tessera-agent` |
| Data persistence, embeddings, pgvector | `tessera-data` |
| Analyst UX, feedback form | `tessera-web` |
| Observability | Langfuse Cloud |
| Infra (dev) | Docker Compose |

## Ports (local dev)

| Service | Port |
|---|---|
| tessera-agent | 8001 |
| tessera-data | 8002 |
| tessera-web | 3000 |
| PostgreSQL | 5432 |
