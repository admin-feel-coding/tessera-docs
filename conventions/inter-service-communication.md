# Inter-Service Communication Contract

Defines how `tessera-agent`, `tessera-data`, and `tessera-web` communicate with each other.

## Authentication

### Dev (local)

Every request between services must include a shared secret in the header:

```
X-Internal-Key: <secret>
```

Each service reads the expected key from its environment:

```bash
# tessera-agent .env
INTERNAL_API_KEY=dev-secret-change-in-prod

# tessera-data .env
INTERNAL_API_KEY=dev-secret-change-in-prod
```

If the header is missing or the key does not match, the receiving service returns `401`.

### Production (GCP)

Services run on Cloud Run, each with its own GCP Service Account. The calling service generates a Google-signed ID token scoped to the target service's URL and sends it as:

```
Authorization: Bearer <google-signed-id-token>
```

The receiving service verifies the token against Google's public keys. No secrets are stored in code or environment variables — GCP manages token lifecycle and rotation automatically.

**IAM rule:** each service account is granted only the `roles/run.invoker` role on the specific target service it needs to call. `tessera-web` → `tessera-agent` only. `tessera-agent` → `tessera-data` only.

### Who calls whom

```
tessera-web  ──►  tessera-agent  (POST /analyze, POST /feedback)
tessera-agent ──►  tessera-data  (GET /cases/similar, GET /users/:id/history, etc.)
tessera-data  ✗    (calls no other service)
```

`tessera-data` has no outbound service calls. It is the data layer — it only receives.

---

## Error Format

All services return errors in the same structure:

```json
{
  "error": {
    "code": "SNAKE_CASE_ERROR_CODE",
    "message": "Human-readable description of what went wrong.",
    "details": {}
  }
}
```

- `code` — machine-readable, uppercase snake case. Used by callers to branch on error type.
- `message` — human-readable. Never expose internal stack traces or database errors.
- `details` — optional object with extra context (e.g. which field failed validation).

### Standard error codes

| HTTP Status | Code | When to use |
|---|---|---|
| 400 | `INVALID_REQUEST` | Malformed body, missing required fields |
| 401 | `UNAUTHORIZED` | Missing or invalid `X-Internal-Key` / Bearer token |
| 404 | `NOT_FOUND` | Resource does not exist |
| 422 | `VALIDATION_ERROR` | Body parses but fails business validation |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error |
| 503 | `SERVICE_UNAVAILABLE` | Dependency (DB, LLM) is unreachable |

### Example

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Case with id 'case_abc123' not found.",
    "details": {
      "resource": "case",
      "id": "case_abc123"
    }
  }
}
```

---

## HTTP Conventions

- **Content-Type:** always `application/json` for both requests and responses.
- **Method semantics:** `GET` for reads (no side effects), `POST` for creates and agent runs, `PATCH` for partial updates.
- **IDs:** use string IDs with a type prefix (`txn_`, `case_`, `user_`). Never expose raw database integer IDs.
- **Timestamps:** ISO 8601 UTC — `2026-06-12T14:30:00Z`.
- **Pagination:** `GET` list endpoints accept `?limit=` and `?cursor=` query params. Default limit: 20. Max: 100.

---

## Timeouts and Retries

| Caller | Target | Timeout | Retries |
|---|---|---|---|
| `tessera-web` | `tessera-agent` | 10s | 0 (no retry — agent runs are not idempotent) |
| `tessera-agent` | `tessera-data` | 3s | 2 (with 100ms backoff) |

The agent has its own 5s end-to-end budget (PRD requirement). Individual tool calls to `tessera-data` must complete in under 3s to leave room for LLM response time.

---

## Health Checks

Every service exposes `GET /health` with no authentication required. Returns `200 OK` with:

```json
{ "status": "ok", "service": "tessera-data", "version": "0.1.0" }
```

Docker Compose and Cloud Run use this endpoint for readiness checks.
