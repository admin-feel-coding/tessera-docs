# Verdict Schema

Every `POST /analyze` response must conform to this schema. The agent is not allowed to return a verdict without a cited source — if it cannot cite one, it must set `decision: ESCALATE`.

## JSON Schema

```json
{
  "transaction_id": "string",
  "decision": "APPROVE | DECLINE | ESCALATE",
  "risk_score": "float (0.0 – 1.0)",
  "reasoning": "string — step-by-step explanation of how the verdict was reached",
  "cited_sources": [
    {
      "type": "rule | case",
      "id": "string — rule name or case ID",
      "excerpt": "string — the specific part of the rule or case that supports the verdict"
    }
  ],
  "signals": {
    "user_history_flag": "boolean",
    "ip_risk_flag": "boolean",
    "device_fingerprint_flag": "boolean",
    "blacklist_hit": "boolean",
    "velocity_flag": "boolean"
  },
  "escalation_reason": "string | null — only set when decision is ESCALATE",
  "latency_ms": "integer",
  "model": "string — model ID used",
  "tool_calls": "integer — number of tool calls made during agent loop",
  "langfuse_trace_id": "string"
}
```

## Grounding Rule

`cited_sources` must contain at least one entry. If the agent cannot cite a rule or retrieved case to support its verdict, it **must** set:

```json
{
  "decision": "ESCALATE",
  "escalation_reason": "No grounded source could be identified to support a verdict.",
  "cited_sources": []
}
```

## Example — Approve

```json
{
  "transaction_id": "txn_abc123",
  "decision": "APPROVE",
  "risk_score": 0.12,
  "reasoning": "Transaction amount ($42) is well within the user's historical average ($38). Device fingerprint matches 14 previous successful transactions. IP is from the user's known home region. Retrieved Case #77 is nearly identical (same device, amount range, merchant category) and was approved by analyst.",
  "cited_sources": [
    {
      "type": "case",
      "id": "case_077",
      "excerpt": "Same device fingerprint, $39 purchase at electronics merchant, approved by analyst on 2026-04-10."
    },
    {
      "type": "rule",
      "id": "rule_known_device_low_amount",
      "excerpt": "Transactions under $100 from a known device with 10+ prior successful purchases are low risk."
    }
  ],
  "signals": {
    "user_history_flag": false,
    "ip_risk_flag": false,
    "device_fingerprint_flag": false,
    "blacklist_hit": false,
    "velocity_flag": false
  },
  "escalation_reason": null,
  "latency_ms": 2340,
  "model": "claude-sonnet-4-6",
  "tool_calls": 3,
  "langfuse_trace_id": "lf_trace_xyz789"
}
```
