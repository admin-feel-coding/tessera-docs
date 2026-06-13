# Code Conventions

## Architecture — 3 Layers (all services)

Every service follows the same mental model:

```
Request → Handler → Service → Store / Client
```

| Layer | Responsibility |
|---|---|
| **Handler** | Parse and validate the HTTP request. Call the service. Return the HTTP response. No business logic. |
| **Service** | Business logic and orchestration. Calls the store or external clients. |
| **Store / Client** | Database queries (Store) or HTTP calls to other services (Client). No business logic. |

If a piece of code does not fit cleanly into one layer, it goes in Service.
No hexagonal architecture, no ports/adapters, no domain/application/infrastructure split.

---

## tessera-agent (Python / FastAPI)

### Folder structure

```
tessera-agent/
├── app/
│   ├── main.py           # FastAPI app, router registration
│   ├── handlers/         # Route handlers — one file per domain
│   ├── services/         # Business logic — agent loop, feedback
│   ├── tools/            # Tool definitions callable by the agent
│   ├── clients/          # HTTP clients for tessera-data
│   └── schemas/          # Pydantic models (requests, responses, verdict)
├── evals/                # Eval suite against golden dataset
├── scripts/              # CLI: seed synthetic data, run evals
├── tests/
├── pyproject.toml
└── .env.example
```

### Tooling

| Tool | Purpose | Config |
|---|---|---|
| `ruff` | Linter + formatter (replaces flake8, black, isort) | `pyproject.toml` |
| `pyright` | Static type checking | `pyproject.toml` |
| `uv` | Package manager and runner | `pyproject.toml` |
| `pytest` | Tests | `pyproject.toml` |

### Rules

- All functions and method parameters must have type annotations.
- Pydantic models for every request body and response — no raw dicts crossing layer boundaries.
- One file per domain in `handlers/` and `services/` (e.g. `analyze.py`, `feedback.py`).
- `ruff check` and `ruff format` must pass before committing.

---

## tessera-data (Go / net/http)

### Folder structure

```
tessera-data/
├── cmd/
│   └── server/
│       └── main.go       # Entry point — wire dependencies, start server
├── internal/
│   ├── handler/          # HTTP handlers — one file per domain
│   ├── service/          # Business logic
│   └── store/            # PostgreSQL queries (pgx)
├── migrations/           # SQL migration files (golang-migrate)
├── tests/
├── go.mod
├── go.sum
└── .env.example
```

### Tooling

| Tool | Purpose |
|---|---|
| `gofmt` | Formatter — built into Go toolchain, non-negotiable |
| `golangci-lint` | Linter |
| `golang-migrate` | Database migrations |
| `pgx/v5` | PostgreSQL driver |
| `chi` | Router (lightweight, stdlib-compatible) |

### Rules

- `gofmt` must pass — no exceptions, enforced by the Go toolchain.
- All exported functions and types must have a one-line doc comment.
- Errors are returned, never swallowed. No `_ = someFunc()` that returns an error.
- No `panic` outside of `main.go` startup.
- Keep `internal/` private — nothing outside this module imports it.

---

## tessera-web (TypeScript / Next.js)

### Folder structure

```
tessera-web/
├── app/
│   ├── page.tsx                   # Queue — pending verdicts list
│   ├── verdicts/
│   │   └── [id]/
│   │       ├── page.tsx           # Verdict detail
│   │       └── feedback/
│   │           └── page.tsx       # Feedback form
│   └── analytics/
│       └── page.tsx               # Metrics dashboard
├── components/                    # Shared UI components (shadcn/ui base)
├── lib/
│   ├── api.ts                     # Typed client for tessera-agent
│   └── utils.ts                   # Shared utilities
└── public/
```

### Tooling

| Tool | Purpose | Config |
|---|---|---|
| `eslint` | Linter | `eslint.config.mjs` |
| `prettier` | Formatter | `.prettierrc` |
| TypeScript | Strict mode (`"strict": true`) | `tsconfig.json` |

### Rules

- Default to **Server Components**. Only add `"use client"` when the component needs interactivity (event handlers, browser APIs).
- Data fetching happens in Server Components or Server Actions — never `useEffect` for initial data.
- All API responses typed with TypeScript interfaces that mirror the verdict schema in `docs/verdict-schema.md`.
- `npm run lint` must pass before committing.

---

## Cross-Service Rules

- **No comments that describe what the code does** — well-named functions and variables do that. Comments only for non-obvious *why*.
- **English only** — all code, comments, variable names, commit messages, and docs.
- **No `console.log` / `print` in production paths** — use structured logging (Python: `structlog`, Go: `slog`, Next.js: server-side `console` is acceptable only in development).
- **`.env.example`** in every service root with all required env vars listed (no values).
