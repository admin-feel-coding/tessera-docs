# tessera-data — Service Context

Go / stdlib net/http service. Owns all data persistence: historical cases, pgvector embeddings, user history, IP/device data, blacklists.

## Stack

- **Runtime:** Go 1.22+
- **Framework:** stdlib `net/http` + `chi` router
- **DB driver:** `pgx/v5`
- **Embeddings:** pgvector via `pgvector-go`
- **Migrations:** `golang-migrate`

## Key Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/cases/similar` | pgvector similarity search (top-k) |
| POST | `/cases` | Insert a new grounded case (from feedback) |
| GET | `/users/:id/history` | User purchase history |
| GET | `/ip/:ip/risk` | IP risk signals |
| GET | `/devices/:id/fingerprint` | Device fingerprint data |
| GET | `/blacklist/check` | Check user/email/card against blacklist |
| GET | `/health` | Health check |

## Database

PostgreSQL 16 + pgvector extension. Schema lives in `migrations/`.

Key tables:
- `cases` — historical fraud decisions with embeddings (vector 1536)
- `users` — synthetic user profiles
- `transactions` — transaction history per user
- `ip_signals` — IP risk data
- `device_fingerprints` — device history
- `blacklist` — blocked users, emails, card BINs

## Environment Variables

```
DATABASE_URL=postgres://tessera:tessera@localhost:5432/tessera
PORT=8002
```

## Dev Commands

```bash
go run ./cmd/server          # start server on :8002
go test ./...                # run tests
migrate -path migrations -database $DATABASE_URL up   # run migrations
```
