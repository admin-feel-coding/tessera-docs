# Tessera — Product Overview

> An AI fraud analyst that investigates ambiguous transactions, calls live data tools, and returns a grounded, cited verdict — or escalates to a human reviewer. Never a guess.

---

## TL;DR

| Question | Answer |
|---|---|
| **What does it do?** | Replaces the human analyst step for the ~5–10% of transactions that rule engines cannot resolve. |
| **Who is it for?** | Fraud ops teams at modern fintech (Stripe, Brex, Ramp, Mercury) drowning in escalation queues. |
| **How does it decide?** | Claude Sonnet 4.6 in a tool-use loop over 6 live data APIs, with an application-layer grounding override that forces ESCALATE when no source is cited. |
| **What does it return?** | A typed `Verdict` — `APPROVE | DECLINE | ESCALATE` — with `cited_sources`, `signals`, `cost_usd`, `latency_ms`, and a Langfuse trace. |
| **SLO** | < 5s per verdict · < $0.10 per decision · ≥ 80% match with human analyst on the golden set. |

---

## 1. The Problem (and why an LLM is the right hammer)

Modern fraud systems do well on the obvious cases. A rule engine catches "card BIN on the blacklist" or "amount > $10k for a new account" instantly. What rule engines can't handle is the **ambiguous middle 5–10%** — the cases where:

- Signals conflict (clean user history + VPN IP)
- Context is required (the merchant category matches the user's prior purchases, but the device is new)
- The pattern is novel (a card-testing attack from an IP that was clean an hour ago)

Today those cases land in an analyst queue. A human reads 3–4 dashboards, looks up case precedent, makes a call. Median time-to-decision: **30+ minutes**. Cost: an analyst at $100k/year reviewing 80–100 cases/day.

Tessera does the same investigation in seconds — but only commits to a decision when it can cite the source. Otherwise it escalates with a categorized reason.

---

## 2. System Architecture (C4 — Context)

The 30-second view: what talks to what at the boundary.

```mermaid
graph TB
    Analyst([👤 Fraud Analyst])
    Rules([⚙️ Upstream Rule Engine])

    subgraph Tessera["Tessera Platform"]
        Web[/"tessera-web<br/>Next.js 15"/]
        Agent["tessera-agent<br/>FastAPI"]
        Data[("tessera-data<br/>Go + Postgres")]
    end

    Claude[/"Anthropic API<br/>Claude Sonnet 4.6"/]
    Langfuse[/"Langfuse Cloud<br/>Traces"/]

    Analyst -->|"submits txn, reviews verdicts"| Web
    Rules -->|"POST /analyze<br/>(ambiguous txn)"| Agent
    Web -->|"SSE stream"| Agent
    Agent <-->|"tool calls + RAG"| Data
    Agent -->|"messages.create<br/>with tools"| Claude
    Agent -->|"spans + generations"| Langfuse

    classDef external fill:#1e1e1e,stroke:#5865f2,color:#e6e6e6
    classDef internal fill:#080808,stroke:#262626,color:#e6e6e6
    class Claude,Langfuse,Rules,Analyst external
    class Web,Agent,Data internal
```

---

## 3. Container View (C4 — Level 2)

The components inside each service.

```mermaid
graph LR
    subgraph WEB["tessera-web · Next.js 15"]
        Landing[Landing<br/>/]
        Analyze[Analyze flow<br/>/analyze]
        Queue[Review queue<br/>/queue]
        Verdict[Verdict detail<br/>/verdicts/:id]
        Analytics[Analytics<br/>/analytics]
        Evals[Evals dashboard<br/>/evals]
        Proxy["/api/* SSE proxy"]
    end

    subgraph AGENT["tessera-agent · FastAPI"]
        AnalyzeH["POST /analyze<br/>POST /analyze/stream"]
        FeedbackH["POST /feedback"]
        EvalsH["GET /evals/latest"]
        VerdictsH["GET /verdicts"]

        Loop[["Agent Loop<br/>(tool-use)"]]
        Grounding{{"Grounding<br/>Override"}}
        Tools["6 tools<br/>dispatch()"]
        Pricing[Cost calc]
    end

    subgraph DATA["tessera-data · Go"]
        UserH[/users/:id/history/]
        IPH[/ip/:ip/risk/]
        DevH[/devices/:id/fingerprint/]
        BlackH[/blacklist/check/]
        CasesH[/cases/similar/]
        VelH[/velocity/check/]
        VerdH[/verdicts/]
        FeedH[/cases POST/]
    end

    PG[(PostgreSQL<br/>+ pgvector)]

    Proxy --> AnalyzeH
    AnalyzeH --> Loop
    FeedbackH --> Tools
    Loop --> Tools
    Loop --> Grounding
    Loop --> Pricing
    Tools --> UserH & IPH & DevH & BlackH & CasesH & VelH
    AnalyzeH --> VerdH
    UserH & IPH & DevH & BlackH & CasesH & VelH & VerdH & FeedH --> PG

    style WEB fill:#0a0a0a,stroke:#5865f2,color:#e6e6e6
    style AGENT fill:#0a0a0a,stroke:#10b981,color:#e6e6e6
    style DATA fill:#0a0a0a,stroke:#f59e0b,color:#e6e6e6
    style PG fill:#1e1e1e,stroke:#262626,color:#e6e6e6
```

---

## 4. Request Lifecycle — Analyze Sequence

What happens between "click Run Analysis" and seeing a verdict.

```mermaid
sequenceDiagram
    autonumber
    actor User as Analyst
    participant Web as tessera-web
    participant Proxy as /api/analyze/stream
    participant Agent as tessera-agent
    participant Claude as Anthropic
    participant Data as tessera-data
    participant LF as Langfuse

    User->>Web: Click "Run Analysis" (preset)
    Web->>Proxy: POST /api/analyze/stream
    Proxy->>Agent: POST /analyze/stream<br/>X-Internal-Key

    Agent->>LF: trace.start(txn_id)
    Agent-->>Web: event: start

    loop Tool-use loop (≤10 iterations)
        Agent->>Claude: messages.create<br/>(tools, history)
        Claude-->>Agent: tool_use block(s)

        Agent-->>Web: event: tool_start
        Agent->>Data: GET /users/.../history<br/>or /ip/.../risk, etc.
        Data-->>Agent: JSON
        Agent-->>Web: event: tool_complete

        Agent->>LF: span.tool(name, latency)
    end

    Claude-->>Agent: stop_reason: end_turn<br/>JSON Verdict

    Note over Agent: Apply grounding override:<br/>if cited_sources < 1 → ESCALATE

    Agent->>Data: POST /verdicts (persist)
    Data-->>Agent: 201 Created
    Agent->>LF: generation.end<br/>(tokens, cost_usd)

    Agent-->>Web: event: verdict
    Agent-->>Web: event: done

    Web->>User: Redirect /verdicts/:id
```

**Latency budget** (target): tools 1.5s · LLM loop 2.5s · grounding override + persist 0.3s · overhead 0.7s = **< 5s** total.

---

## 5. The Agent Loop (flowchart)

The decision logic inside `tessera-agent/app/services/analyze.py`.

```mermaid
flowchart TD
    Start([Transaction arrives]) --> Init[Build system prompt<br/>+ user message]
    Init --> Call[Call Claude with 6 tools]
    Call --> Stop{stop_reason?}

    Stop -->|tool_use| Tools[Execute tool calls<br/>via dispatch]
    Tools --> Cap{tool_calls<br/>≥ 10?}
    Cap -->|yes| ForceEsc[Force ESCALATE<br/>category: LOW_CONFIDENCE]
    Cap -->|no| Append[Append tool_result<br/>blocks to messages]
    Append --> Call

    Stop -->|end_turn| Parse[Extract JSON from<br/>final_text]
    Parse --> Valid{Validates<br/>against Verdict<br/>schema?}
    Valid -->|no| FailEsc[Force ESCALATE<br/>category: NOVEL_PATTERN]
    Valid -->|yes| Override{cited_sources<br/>≥ 1?}

    Override -->|no| GroundEsc[Force ESCALATE<br/>category: INSUFFICIENT_GROUNDING]
    Override -->|yes| Final[Verdict ready]

    ForceEsc --> Persist
    FailEsc --> Persist
    GroundEsc --> Persist
    Final --> Persist[POST /verdicts]
    Persist --> Stream[Emit event: verdict]
    Stream --> Done([event: done])

    classDef ok fill:#0a3d1e,stroke:#10b981,color:#e6e6e6
    classDef warn fill:#3d2f0a,stroke:#f59e0b,color:#e6e6e6
    classDef bad fill:#3d0a0a,stroke:#ef4444,color:#e6e6e6
    class Final,Done ok
    class ForceEsc,FailEsc,GroundEsc warn
    class Override,Valid,Cap bad
```

The three forced-escalation paths are the architectural insight: **Tessera never silently fails**. Every uncited verdict, every parse failure, every runaway loop becomes an ESCALATE with a typed category.

---

## 6. Verdict State Machine (frontend perspective)

What the browser tracks during a single analyze flow.

```mermaid
stateDiagram-v2
    [*] --> idle: page load
    idle --> streaming: click "Run Analysis"
    streaming --> streaming: tool_start / tool_complete
    streaming --> done: event: verdict
    streaming --> error: event: error<br/>or stream closed
    done --> [*]: redirect /verdicts/:id
    error --> idle: user retries

    state streaming {
        [*] --> connecting
        connecting --> running: event: start
        running --> running: append tool event
    }
```

---

## 7. Data Model (ER diagram)

The Postgres schema — what is persisted between requests.

```mermaid
erDiagram
    TRANSACTIONS {
        UUID id PK
        STRING user_id FK
        DECIMAL amount
        STRING currency
        STRING status
        INET ip_address
        VARCHAR card_bin
        VARCHAR device_id
        TIMESTAMP created_at
    }

    VERDICTS {
        UUID id PK
        STRING transaction_id
        STRING decision
        FLOAT risk_score
        TEXT reasoning
        JSONB cited_sources
        JSONB signals
        VARCHAR escalation_category
        TEXT escalation_reason
        INT latency_ms
        STRING model
        INT tool_calls
        INT input_tokens
        INT output_tokens
        NUMERIC cost_usd
        STRING langfuse_trace_id
        TIMESTAMP created_at
    }

    CASES {
        UUID id PK
        STRING transaction_id
        TEXT summary
        STRING analyst_decision
        VECTOR embedding "1536-dim"
        JSONB key_signals
        TIMESTAMP created_at
    }

    USERS {
        STRING id PK
        INT total_transactions
        INT chargeback_count
        FLOAT avg_amount
        INT account_age_days
    }

    BLACKLIST_ENTRIES {
        UUID id PK
        STRING match_type
        STRING value
        STRING source
        TIMESTAMP added_at
    }

    IP_SIGNALS {
        INET ip_address PK
        FLOAT risk_score
        BOOL vpn_detected
        BOOL tor_detected
        STRING country_code
        TIMESTAMP last_updated
    }

    TRANSACTIONS ||--o{ VERDICTS : "1 → many"
    VERDICTS ||--o| CASES : "feedback → case"
    USERS ||--o{ TRANSACTIONS : "owns"
    TRANSACTIONS }o--|| IP_SIGNALS : "ip lookup"
```

---

## 8. The RAG Flywheel (data flow)

The most subtle part of the product — how analyst feedback becomes future agent context.

```mermaid
flowchart LR
    T1[New txn] --> A1[Agent analyzes]
    A1 --> V1[Verdict v1]
    V1 --> Q[Review queue]
    Q --> R{Analyst<br/>review}

    R -->|confirms| Done1([Done])
    R -->|corrects| F[POST /feedback]

    F --> E[Embed<br/>case summary]
    E --> S[(pgvector store)]

    S -.->|search_similar_cases| A2[Next agent run]
    A2 --> V2[Verdict v2<br/>cites case_id]

    style S fill:#0a1f3d,stroke:#5865f2,color:#e6e6e6
    style E fill:#0a3d1e,stroke:#10b981,color:#e6e6e6
```

Every correction feeds the corpus. Within a few weeks, the agent has retrieved precedent for most of its decisions — and the grounding override means it cites them rather than guessing.

---

## 9. Tool Catalog

The 6 tools the agent has at its disposal. Order matters — the descriptions in `tessera-agent/app/tools/__init__.py` give Claude sequencing hints.

| # | Tool | Endpoint | Returns | When to call |
|---|---|---|---|---|
| 1 | `get_user_history` | `GET /users/:id/history` | total_txns, chargeback_count, account_age_days | First, every analysis — establishes baseline |
| 2 | `get_ip_risk` | `GET /ip/:ip/risk` | risk_score, vpn/tor flags, ISP | Early — drives later escalation choices |
| 3 | `get_device_fingerprint` | `GET /devices/:id/fingerprint` | seen_before, fraud_flag | After user history — detects user/device mismatch |
| 4 | `check_blacklist` | `GET /blacklist/check` | matched (bool), match_type, source | Every transaction — fast DECLINE path |
| 5 | `search_similar_cases` | `POST /cases/similar` | top-5 cases with similarity_score | When signals conflict — grounded precedent |
| 6 | `check_velocity` | `GET /velocity/check` | distinct_users_by_ip, by_bin, total_in_window | When IP risk is elevated — card-testing detector |

**Tool cap**: 10 calls per analysis. Reaching the cap forces an ESCALATE with category `LOW_CONFIDENCE` — the agent gave up rather than guessing.

---

## 10. Verdict Anatomy

Every analysis produces this exact shape. The grounding contract is enforced at the application layer (Pydantic + a post-validation check), not just in the prompt.

```jsonc
{
  "transaction_id": "txn_demo_fraud_001",
  "decision": "DECLINE",                          // APPROVE | DECLINE | ESCALATE
  "risk_score": 0.92,
  "reasoning": "Card BIN 424242 matches blacklist...",
  "cited_sources": [                              // ← grounding contract
    { "type": "blacklist", "id": "bl_424242", "added_at": "2026-05-01T..." },
    { "type": "case",      "id": "case_a1b2", "similarity": 0.91 }
  ],
  "signals": [
    { "name": "blacklist_match",  "severity": "high",   "value": "424242" },
    { "name": "velocity_flag",    "severity": "high",   "value": "8 users/BIN in 60min" },
    { "name": "ip_risk",          "severity": "medium", "value": 0.78 }
  ],
  "escalation_category": null,                    // typed enum (5 values) or null
  "escalation_reason":   null,                    // human-readable detail
  "latency_ms": 4312,
  "model": "claude-sonnet-4-6",
  "tool_calls": 5,
  "input_tokens": 12847,                          // ← unit economics
  "output_tokens": 312,
  "cost_usd": 0.04323,
  "langfuse_trace_id": "lf_8f3a1c..."            // ← full reasoning trace
}
```

**The grounding contract:** if `cited_sources.length < 1` AND `decision != "ESCALATE"`, the application layer rewrites the verdict to `ESCALATE` with `escalation_category = "INSUFFICIENT_GROUNDING"`. This happens in code, not in the prompt — Claude cannot bypass it.

---

## 11. Observability & Unit Economics

### Cost per decision

Tessera tracks every Anthropic call's `input_tokens` and `output_tokens` across the full tool-use loop and computes `cost_usd` per verdict at pricing constants for Claude Sonnet 4.6 (`$3.00/M in`, `$15.00/M out`). The analytics page surfaces:

```mermaid
xychart-beta
    title "Cost per verdict — last 100 runs (target < $0.10)"
    x-axis [1, 25, 50, 75, 100]
    y-axis "$ USD" 0.0 --> 0.15
    bar [0.038, 0.042, 0.045, 0.039, 0.051]
    line [0.10, 0.10, 0.10, 0.10, 0.10]
```

### Latency budget

```mermaid
pie showData
    title "Where the 5s latency budget goes"
    "Tool calls (data service)" : 30
    "LLM loop (Claude)" : 50
    "Grounding override + persist" : 6
    "SSE framing + overhead" : 14
```

### Langfuse spans

Every verdict has one parent trace with child spans for each tool call and each `messages.create`. Engineers reproduce any production decision by opening the trace.

---

## 12. Evaluation Methodology

### The golden set

50–100 cases, each labeled `expected_decision` by a human analyst. Stored in `tessera-agent/evals/golden_dataset.json`. The CLI runs the eval against the live agent service (not a mock) and writes results to `evals/results/latest.json`.

```bash
cd tessera-agent && uv run python -m evals --repeats 3
```

### What gets measured

| Metric | Definition | Surfaced on |
|---|---|---|
| **match_rate** | fraction of cases where actual == expected | `/evals` headline |
| **precision (DECLINE)** | of N declines, how many were truly fraud | `/analytics` Calibration |
| **recall (DECLINE)** | of N true frauds, how many we declined | `/analytics` Calibration |
| **pass^k** | fraction of cases where ALL k runs were correct | `/evals` Reliability card |
| **avg / p95 latency** | end-to-end ms | `/evals` overview |
| **errors** | runs that raised | `/evals` overview |
| **confusion matrix** | 3×3 expected vs actual | `/evals` heatmap |

### Why pass^k matters

A single match_rate of 96% sounds great — until you realize that for a typical case, your agent has a 4% chance of being wrong on any given run. Pass^3 forces all 3 independent runs to be correct, which is the metric that maps to production reliability. **Always compare pass^k, not match_rate.**

---

## 13. Tech Stack

```mermaid
graph LR
    subgraph LLM["AI Layer"]
        Sonnet[Claude Sonnet 4.6<br/>tool use + structured output]
    end

    subgraph Compute["Compute"]
        Py[Python 3.12<br/>FastAPI + httpx + Pydantic]
        Go[Go 1.22<br/>chi + pgx v5]
        TS[TypeScript<br/>Next.js 15 + Tailwind v4]
    end

    subgraph Data["Data"]
        PG[(PostgreSQL 16)]
        PGV[pgvector<br/>1536-dim, ivfflat]
    end

    subgraph Obs["Observability"]
        LF[Langfuse Cloud<br/>traces + generations]
        Struct[structlog<br/>JSON logs]
    end

    subgraph Infra["Infra (dev)"]
        DC[Docker Compose]
    end

    LLM --> Compute
    Compute --> Data
    Compute --> Obs
    Compute --> Infra
```

---

## 14. Service Surface Area

The contract every service exposes.

### `tessera-agent` (port 8001)

| Method | Path | Purpose |
|---|---|---|
| POST | `/analyze` | One-shot verdict (blocking) |
| POST | `/analyze/stream` | SSE stream of `tool_start` / `tool_complete` / `verdict` |
| POST | `/feedback` | Index an analyst correction into the RAG corpus |
| GET | `/verdicts` | List recent verdicts (proxies to data) |
| GET | `/evals/latest` | Latest eval run JSON |
| GET | `/healthz` | Liveness |

### `tessera-data` (port 8002)

| Method | Path | Purpose |
|---|---|---|
| GET | `/users/:id/history` | Tool 1 backend |
| GET | `/ip/:ip/risk` | Tool 2 backend |
| GET | `/devices/:id/fingerprint` | Tool 3 backend |
| GET | `/blacklist/check` | Tool 4 backend |
| POST | `/cases/similar` | Tool 5 backend (pgvector) |
| GET | `/velocity/check` | Tool 6 backend |
| POST | `/cases` | Feedback ingest (embed + insert) |
| POST | `/verdicts` | Persist verdict + upsert txn velocity fields |
| GET | `/verdicts` | List for queue / analytics |
| GET | `/verdicts/:transaction_id` | Verdict detail |

All authenticated via `X-Internal-Key`. All errors follow the `{"error":{"code","message","details"}}` envelope.

### `tessera-web` (port 3000)

| Route | Purpose |
|---|---|
| `/` | Landing page |
| `/analyze` | Submit a transaction; live SSE panel |
| `/queue` | Recent verdicts |
| `/verdicts/:id` | Verdict detail + feedback form |
| `/analytics` | Decision distribution, calibration, escalation breakdown |
| `/evals` | Live eval results |

Same-origin API routes under `/api/*` proxy SSE and JSON to the agent, keeping the internal key off the browser.

---

## 15. The Trust Contract (why a CTO should believe this)

Three architectural choices distinguish Tessera from "Claude + prompt + ship":

1. **Grounding is enforced in code, not in the prompt.** The Pydantic schema validates structure; a post-validation hook validates citation. Uncited verdicts cannot escape — they are rewritten to `ESCALATE` deterministically.

2. **ESCALATE is a first-class output, not an error.** The agent has explicit permission to say "I don't know" via 5 typed categories (`CONFLICTING_SIGNALS`, `INSUFFICIENT_GROUNDING`, `LOW_CONFIDENCE`, `NOVEL_PATTERN`, `POLICY_REQUIRED`). Analytics surfaces the breakdown so the ops team can tune the system.

3. **Unit economics are tracked, not assumed.** Token usage is accumulated across the tool-use loop and persisted per verdict. The analytics page colors the average cost red if it crosses $0.10. The team knows immediately if a prompt change pushed the SLO.

---

## 16. Roadmap

```mermaid
gantt
    title Tessera Build Roadmap (2026)
    dateFormat YYYY-MM-DD
    axisFormat %b
    section Phase 1
        Data service + schema           :done,    p1a, 2026-03-01, 14d
        Agent loop + tools              :done,    p1b, after p1a, 21d
    section Phase 2
        Web dashboard MVP               :done,    p2a, 2026-04-01, 14d
        Langfuse tracing                :done,    p2b, after p2a, 7d
    section Phase 3
        Linear redesign + SSE streaming :done,    p3a, 2026-05-01, 21d
        Rate limiting + mobile          :done,    p3b, after p3a, 7d
    section Phase 4 (current)
        Live evals + precision/recall   :done,    p4a, 2026-06-10, 3d
        Cost tracking + velocity tool   :done,    p4b, after p4a, 4d
        Feedback loop + categories      :done,    p4c, after p4b, 3d
    section Phase 5
        Real embedding API              :         p5a, 2026-06-20, 7d
        Cross-user fraud network        :         p5b, after p5a, 14d
        GCP deployment                  :         p5c, after p5b, 14d
    section Phase 6
        Multi-tenant + auth             :         p6a, 2026-08-01, 21d
        Public API + SDK                :         p6b, after p6a, 14d
```

---

## 17. Quick Start (local dev)

```bash
# Infra
docker compose up -d                               # Postgres + pgvector

# Data service
cd tessera-data && go run ./cmd/server &           # :8002

# Agent service
cd tessera-agent && uv run fastapi dev --port 8001 &

# Web
cd tessera-web && npm run dev                      # :3000

# Run an eval
cd tessera-agent && uv run python -m evals --repeats 3

# Open the dashboard
open http://localhost:3000
```

Mock mode runs end-to-end without an `ANTHROPIC_API_KEY` — synthetic tool delays, deterministic verdicts, full SSE stream. Useful for demos and CI.

---

## 18. Where to look next

| You want… | Read |
|---|---|
| The original product spec | [`prd.md`](./prd.md) |
| The verdict JSON contract | [`verdict-schema.md`](./verdict-schema.md) |
| Why microservices over monolith | [`decisions/ADR-001-microservices-over-monolith.md`](./decisions/ADR-001-microservices-over-monolith.md) |
| Why this language for this service | [`decisions/ADR-002-language-selection.md`](./decisions/ADR-002-language-selection.md) |
| Why grounding lives in code, not the prompt | [`decisions/ADR-003-grounding-enforcement.md`](./decisions/ADR-003-grounding-enforcement.md) |
| Why RAG over fine-tuning | [`decisions/ADR-004-rag-over-fine-tuning.md`](./decisions/ADR-004-rag-over-fine-tuning.md) |
| Auth, error format, timeouts between services | [`conventions/inter-service-communication.md`](./conventions/inter-service-communication.md) |
| The Langfuse contract and logging shape | [`conventions/observability.md`](./conventions/observability.md) |
| Code structure per service | [`conventions/code-conventions.md`](./conventions/code-conventions.md) |
| Service-specific context for an agent/IDE | [`agent_docs/`](./agent_docs/) |
