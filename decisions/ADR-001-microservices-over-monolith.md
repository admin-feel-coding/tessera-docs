# ADR-001: Microservices over Modular Monolith

## Status
Accepted

## Context
Tessera is a greenfield project with three clearly distinct responsibilities: AI agent reasoning, data persistence with vector search, and an analyst-facing UI. Each responsibility has a different runtime profile — the agent is I/O-bound and LLM-latency-dominated, the data service is query-bound, and the web layer is user-facing. Beyond technical fit, the project serves a dual purpose: demonstrating production-grade system design across multiple modern stacks.

## Decision
Structure Tessera as three independent services, each with its own language, runtime, and deployment unit:

- `tessera-agent` — Python / FastAPI (AI agent, tools, RAG)
- `tessera-data` — Go / net/http (data persistence, pgvector)
- `tessera-web` — TypeScript / Next.js (analyst dashboard)

Services communicate over HTTP. A shared Docker Compose file manages local infrastructure (PostgreSQL + pgvector).

## Alternatives Considered

**Modular monolith (NestJS):** Would be faster to bootstrap and simpler to debug locally. Rejected because it collapses all three responsibilities into a single runtime, prevents independent scaling, and limits the stack diversity that is an explicit goal of this project.

**Monorepo with shared packages:** Reduces duplication across services but adds tooling overhead (Nx, Turborepo) and coupling risk. Rejected in favor of fully independent repos per service — simpler, cleaner boundaries, and each service is self-contained.

## Consequences

- Each service can be deployed, scaled, and versioned independently.
- Inter-service calls add network latency — acceptable given the agent's latency budget (< 5s) is dominated by LLM response time, not HTTP hops.
- Local dev requires Docker Compose to coordinate infrastructure, and each service must be started independently.
- The stack diversity makes this project a strong portfolio artifact demonstrating Python, Go, and TypeScript in a single coherent system.
- Isolation means bugs in one service do not crash others — a meaningful property for a fraud decisioning system.
