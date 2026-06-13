# ADR-002: Language Selection Per Service

## Status
Accepted

## Context
Tessera is a three-service system with distinct runtime profiles: an AI agent loop, a data persistence layer, and an analyst-facing UI. Each service has different performance characteristics and ecosystem requirements. An explicit goal of the project is to demonstrate proficiency across multiple modern stacks targeting the kind of companies — modern fintechs and startups — that favor lean, cloud-native runtimes over heavyweight enterprise frameworks.

## Decision

### tessera-agent → Python 3.12 + FastAPI

Python is the native language of the AI/LLM ecosystem. Anthropic ships features to its Python SDK first — tool use, streaming, batch API, and prompt caching all land in Python before any other SDK. The broader ecosystem (pgvector clients, embedding libraries, eval frameworks, Langfuse SDK) has first-class Python support. FastAPI is the industry standard for ML/AI backend services — it is what Anthropic, OpenAI, and most AI-native startups use to expose their agent backends.

**The Vercel AI SDK (TypeScript) was considered** but rejected for this service. It is an excellent library for frontend streaming (showing agent reasoning token-by-token in a Next.js UI), not for building a backend agent loop with tool use, RAG, and observability. It will be used in `tessera-web` for that purpose if streaming is needed.

### tessera-data → Go 1.22 + stdlib net/http

Go is the dominant language in modern fintech infrastructure — Stripe, Mercado Libre, and most cloud-native payment companies run their data services in Go. It compiles to a single binary, starts in milliseconds (critical for containerized deployments), has a low memory footprint, and its concurrency model (goroutines) is well-suited for handling concurrent data queries from the agent loop. The standard library's `net/http` is production-grade on its own; `chi` is added only for routing ergonomics.

**Java (Spring Boot) was considered** and rejected. While heavily demanded in traditional banking and enterprise fintech, it targets a different employer profile than this project aims for. The JVM's startup time and memory overhead add friction in containerized environments, and Spring Boot's abstraction layer increases cognitive overhead for a service that is fundamentally a thin data API over PostgreSQL.

### tessera-web → TypeScript + Next.js 15 App Router

Next.js App Router is the current standard for React applications that need both server-rendered pages and interactive client components. The analyst dashboard requires server-side data fetching (verdict lists), interactive client components (feedback form), and potentially streaming (showing agent reasoning). Next.js handles all three in a single framework with minimal configuration. TypeScript strict mode enforces the verdict schema contract at the UI boundary.

## Alternatives Considered

| Option | Rejected because |
|---|---|
| NestJS for all services | Collapses stack diversity; strong familiarity bias — this project is also about learning |
| TypeScript/Node for tessera-agent | Anthropic Python SDK ships features first; AI/ML ecosystem is Python-native |
| Java/Spring Boot for tessera-data | Targets enterprise banking, not modern fintech startups; JVM overhead in containers |
| Rust for tessera-data | Steep learning curve outweighs performance gains for a CRUD + vector search service |

## Consequences

- The stack maps directly to the ecosystem of modern fintech startups — the intended career target.
- Python and Go are the two most in-demand backend languages at AI-native and fintech companies respectively.
- Each service requires separate local tooling (uv for Python, Go toolchain, Node for Next.js) — Docker Compose handles the shared infra, but developers need all three runtimes installed.
- The Anthropic Python SDK guarantees access to the latest Claude features without waiting for ports to other SDKs.
