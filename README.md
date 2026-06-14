# tessera-docs

Documentation for **Tessera** — an AI fraud analyst agent that investigates ambiguous transactions, calls 6 live data tools, and returns a grounded, cited verdict — or escalates to a human reviewer.

---

## What is Tessera?

Tessera is a portfolio project demonstrating how to build a production-grade AI agent on top of Claude Sonnet 4.6. It replaces the role of a human fraud analyst for ambiguous transactions that rule engines can't resolve.

Given a transaction, the agent:
1. Calls up to 6 data tools (user history, IP risk, device fingerprint, blacklist, similar cases, velocity)
2. Reasons over the aggregated signals
3. Returns an **APPROVE**, **DECLINE**, or **ESCALATE** verdict — with cited sources and a full reasoning chain
4. Falls back to ESCALATE if it cannot cite at least one grounded source (enforced at the application layer, not just in the prompt)

**Stack:** Claude Sonnet 4.6 · FastAPI (Python) · Go (stdlib) · Next.js 15 · PostgreSQL + pgvector · Langfuse

---

## Repository layout

```
tessera-docs/
├── product-overview.md        # Full product overview with architecture diagrams (start here)
├── architecture.md            # Service map, data flow, inter-service contracts
├── prd.md                     # Product Requirements Document (v1) — goals, success criteria, constraints
├── verdict-schema.md          # The JSON contract every agent run must produce
│
├── agent_docs/
│   ├── tessera-agent.md       # Python/FastAPI agent service — tools, streaming, grounding
│   ├── tessera-data.md        # Go data API — endpoints, store layer, velocity checks
│   └── tessera-web.md         # Next.js frontend — analyze flow, SSE streaming, scenario gallery
│
├── conventions/
│   ├── code-conventions.md    # Folder structure, tooling, naming per service
│   ├── inter-service-communication.md  # Auth, error format, HTTP rules, timeouts
│   └── observability.md       # Langfuse tracing contract, structured logging
│
└── decisions/
    ├── ADR-001-microservices-over-monolith.md
    ├── ADR-002-language-selection.md
    ├── ADR-003-grounding-enforcement.md
    └── ADR-004-rag-over-fine-tuning.md
```

---

## Where to start

| Goal | Document |
|---|---|
| Understand the product | [product-overview.md](product-overview.md) |
| Understand the system design | [architecture.md](architecture.md) |
| Understand the agent's decision contract | [verdict-schema.md](verdict-schema.md) |
| Read the requirements | [prd.md](prd.md) |
| Work on the agent service | [agent_docs/tessera-agent.md](agent_docs/tessera-agent.md) |
| Work on the data API | [agent_docs/tessera-data.md](agent_docs/tessera-data.md) |
| Work on the frontend | [agent_docs/tessera-web.md](agent_docs/tessera-web.md) |
| Understand a design decision | [decisions/](decisions/) |

---

## Key design decisions

| # | Decision | Rationale |
|---|---|---|
| ADR-001 | Microservices over monolith | Python for AI/LLM, Go for data API, TypeScript for UI — each language owns its domain |
| ADR-002 | Per-language service boundaries | Avoid the "one language to rule them all" trap; use the right tool per layer |
| ADR-003 | Strict grounding enforcement | Uncited verdicts auto-escalate — enforced in code, not in the prompt |
| ADR-004 | RAG over fine-tuning | Feedback loop updates the corpus in real time; fine-tuning requires offline retraining |

---

## Services

| Service | Language | Port | Responsibility |
|---|---|---|---|
| `tessera-agent` | Python · FastAPI | 8001 | LLM orchestration, tool dispatch, SSE streaming, grounding |
| `tessera-data` | Go · net/http | 8002 | User data, IP risk, device fingerprint, blacklist, velocity, verdict persistence |
| `tessera-web` | TypeScript · Next.js 15 | 3000 | Analyze UI, queue, analytics, evals dashboard |
| Postgres + pgvector | — | 5432 | Verdicts, cases (RAG corpus), transactions, blacklist |

---

## Running the project

From the `tessera-automation/` directory:

```bash
make dev       # start all services + seed the database
make seed      # reseed only
make test      # run all quality gates (ruff, go vet, npm build)
make stop      # stop Postgres
```

To run the eval suite against the golden dataset:

```bash
cd tessera-agent
uv run python -m evals
# Results are written to evals/results/latest.json
# and surfaced live at http://localhost:3000/evals
```

---

## Success criteria (v1)

- ≥ 80 % match rate with human analyst on the 50-case golden dataset
- < 5 s per verdict (p95)
- < $0.10 per decision (Claude Sonnet 4.6 pricing)
- Every verdict cites at least one grounded source — or escalates
