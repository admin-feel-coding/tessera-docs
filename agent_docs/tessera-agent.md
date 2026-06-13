# tessera-agent — Service Context

Python / FastAPI service. The core of Tessera — runs the agent loop, calls Claude, enforces grounding, writes Langfuse traces.

## Stack

- **Runtime:** Python 3.12+, managed with `uv`
- **Framework:** FastAPI
- **AI:** Anthropic Python SDK (`anthropic`)
- **Observability:** Langfuse Python SDK
- **Validation:** Pydantic v2
- **HTTP client:** `httpx` (for calling tessera-data)

## Key Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/analyze` | Run agent on a transaction, return verdict |
| POST | `/feedback` | Forward analyst correction to tessera-data |
| GET | `/health` | Health check |

## Agent Loop Pattern

The agent uses the Anthropic tool-use API in a loop:
1. Send transaction + system prompt to Claude
2. If Claude returns tool_use blocks → execute tools → append results → loop
3. When Claude returns a text response → parse as verdict JSON → validate with Pydantic
4. If cited_sources is empty → override decision to ESCALATE

Max tool calls per run: 10 (guardrail).

## Tools Available to the Agent

- `get_user_history(user_id)` → calls tessera-data
- `get_ip_risk(ip_address)` → calls tessera-data
- `get_device_fingerprint(device_id)` → calls tessera-data
- `check_blacklist(user_id, email, card_bin)` → calls tessera-data
- `search_similar_cases(transaction)` → pgvector similarity search via tessera-data

## Environment Variables

```
ANTHROPIC_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
TESSERA_DATA_URL=http://localhost:8002
```

## Dev Commands

```bash
uv run fastapi dev          # hot-reload dev server on :8001
uv run pytest               # run tests
uv run python -m scripts.seed_synthetic  # generate + seed synthetic dataset
```
