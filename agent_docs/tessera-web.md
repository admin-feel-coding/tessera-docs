# tessera-web — Service Context

Next.js 15 App Router frontend. Analyst-facing dashboard for reviewing agent verdicts and submitting feedback.

## Stack

- **Framework:** Next.js 15 + App Router
- **Language:** TypeScript (strict mode)
- **Styling:** Tailwind CSS v4
- **Components:** shadcn/ui
- **State:** React Server Components + minimal client state
- **API calls:** Server Actions for mutations, fetch in RSC for reads

## Key Views

| Route | Description |
|---|---|
| `/` | Queue — list of pending verdicts to review |
| `/verdicts/:id` | Verdict detail — reasoning, signals, cited sources |
| `/verdicts/:id/feedback` | Feedback form — confirm/correct + note |
| `/analytics` | Metrics dashboard — accuracy, latency, cost |

## Environment Variables

```
TESSERA_AGENT_URL=http://localhost:8001
```

## Dev Commands

```bash
npm run dev       # start on :3000
npm run build     # production build
npm run lint      # eslint
```

## Design Principles

- Analysts confirm or edit what the agent already surfaced — they do not start from a blank form.
- Verdict reasoning and cited sources are always visible before the feedback form.
- Feedback must be low-friction: 2 clicks for agreement, short note for corrections.
