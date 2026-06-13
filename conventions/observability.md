# Observability Contract

Defines what every service must trace so that Langfuse shows a coherent, end-to-end picture of every agent run.

## Core Concept: One Trace Per Analysis Request

When `tessera-agent` receives `POST /analyze`, it creates one **Trace** in Langfuse. That trace owns every span produced during the entire request — tool calls, DB queries, the Claude API call, and verdict validation. Everything that happens to resolve one transaction belongs to one trace.

```
TRACE: analyze_transaction (txn_abc123)
├── SPAN:       tool.get_user_history         89ms
├── SPAN:       tool.check_blacklist          45ms
├── SPAN:       tool.search_similar_cases    112ms
├── GENERATION: claude.completion           2,840ms  →  1,847 input / 312 output tokens / $0.031
└── SPAN:       verdict.validate               3ms
```

---

## Trace ID Flow

`tessera-agent` generates the `trace_id` when the request arrives. It is the authority for the trace.

```
tessera-web
  └── POST /analyze
        │
        ▼
tessera-agent  ← creates Langfuse trace, generates trace_id
  └── GET /users/:id/history
        │  header: X-Trace-ID: <trace_id>
        ▼
tessera-data   ← logs the trace_id on every DB query for correlation
```

The `trace_id` is returned in every verdict response as `langfuse_trace_id`. `tessera-web` uses it to build a direct link to the Langfuse dashboard for that decision.

---

## What tessera-agent Must Trace

`tessera-agent` owns the trace. It is responsible for recording everything.

### Trace (one per request)

```python
trace = langfuse.trace(
    name="analyze_transaction",
    input={"transaction_id": txn_id, "amount": ..., "user_id": ...},
    metadata={"service": "tessera-agent"},
)
```

### Span — each tool call

```python
span = trace.span(
    name="tool.get_user_history",   # always prefixed with tool.
    input={"user_id": user_id},
)
# ... execute tool ...
span.end(output={"purchase_count": 14, "avg_amount": 38.5})
```

Tool span names:

| Tool | Span name |
|---|---|
| get_user_history | `tool.get_user_history` |
| get_ip_risk | `tool.get_ip_risk` |
| get_device_fingerprint | `tool.get_device_fingerprint` |
| check_blacklist | `tool.check_blacklist` |
| search_similar_cases | `tool.search_similar_cases` |

### Generation — Claude API call

Use `trace.generation()` (not `trace.span()`) for the Claude call. Langfuse uses this to track token usage and cost automatically.

```python
generation = trace.generation(
    name="claude.completion",
    model="claude-sonnet-4-6",
    input=messages,            # the full messages array sent to Claude
)
# ... call Claude API ...
generation.end(
    output=response.content,
    usage={
        "input": response.usage.input_tokens,
        "output": response.usage.output_tokens,
    },
)
```

### Trace end — after verdict is validated

```python
trace.update(
    output={
        "decision": verdict.decision,
        "risk_score": verdict.risk_score,
        "cited_sources_count": len(verdict.cited_sources),
        "tool_calls": verdict.tool_calls,
    },
    metadata={"escalated": verdict.decision == "ESCALATE"},
)
```

---

## What tessera-data Must Do

`tessera-data` does not create Langfuse traces — it has no Langfuse dependency. It only needs to:

1. Read `X-Trace-ID` from every incoming request header.
2. Include it in structured log output for every DB query.

```go
// Every log line from tessera-data includes trace_id
slog.Info("query executed",
    "trace_id", r.Header.Get("X-Trace-ID"),
    "query",    "search_similar_cases",
    "latency_ms", elapsed.Milliseconds(),
    "rows",     rowCount,
)
```

This means when you search Langfuse or GCP Logs for a `trace_id`, you see both the agent spans and the DB query logs side by side.

---

## What tessera-web Must Do

`tessera-web` does not create traces. It must:

1. Display the `langfuse_trace_id` from the verdict as a link to the Langfuse dashboard on the verdict detail page.
2. Log errors server-side with `console.error` including the `transaction_id`.

---

## Structured Log Format (all services)

Every log line must be structured JSON. No plain string logs in production.

```json
{
  "level": "info",
  "timestamp": "2026-06-12T14:30:00Z",
  "service": "tessera-agent",
  "trace_id": "lf_trace_xyz789",
  "transaction_id": "txn_abc123",
  "message": "verdict produced",
  "decision": "APPROVE",
  "latency_ms": 3210
}
```

| Field | Required | Description |
|---|---|---|
| `level` | Yes | `debug`, `info`, `warn`, `error` |
| `timestamp` | Yes | ISO 8601 UTC |
| `service` | Yes | Service name |
| `trace_id` | When available | Langfuse trace ID |
| `transaction_id` | When available | The transaction being analyzed |
| `message` | Yes | What happened |

### Loggers by service

| Service | Library |
|---|---|
| `tessera-agent` | `structlog` (Python) |
| `tessera-data` | `slog` (Go stdlib, built-in since Go 1.21) |
| `tessera-web` | `console` (server-side only, Next.js) |

---

## What to Trace vs What to Log

| | Langfuse trace/span | Structured log |
|---|---|---|
| Claude API call | Yes — Generation span | No (already in Langfuse) |
| Tool call | Yes — Span | No |
| DB query | No | Yes (with trace_id) |
| HTTP request received | No | Yes |
| Verdict produced | Yes — trace.update() | Yes |
| Unhandled error | Yes — span.end(level="ERROR") | Yes |

---

## Environment Variables (tessera-agent)

```
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```
