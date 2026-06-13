# ADR-003: Strict Grounding Enforcement — Cite a Source or Escalate

## Status
Accepted

## Context
LLM-based agents can produce confident-sounding verdicts that are not grounded in any actual evidence — they draw on general training knowledge rather than project-specific rules or historical cases. In a fraud decisioning context, this is dangerous: an analyst reviewing the verdict has no way to distinguish a well-reasoned conclusion from a hallucinated one, and may act on it without question.

Tessera is explicitly designed for the ambiguous tier of transactions — cases where the evidence is thin by definition. This makes grounding enforcement more critical, not less. An agent that fabricates confidence when evidence is absent is worse than no agent at all.

## Decision
Every verdict must cite at least one grounded source before a decision of APPROVE or DECLINE is issued. A grounded source is either:

- A **named business rule** defined in the system prompt by the owner, or
- A **retrieved historical case** from the pgvector store, surfaced via similarity search.

If the agent completes its tool loop and cannot cite at least one source of either type, it must return `decision: ESCALATE` with a clear `escalation_reason`. The verdict is not allowed to contain an APPROVE or DECLINE with an empty `cited_sources` array — this is enforced at the Pydantic validation layer in `tessera-agent`, not left to the model's discretion.

The escalation is not a failure state. It is the correct output when evidence is insufficient. The human analyst resolves the case, and that resolution — with the analyst's note and decisive signals — is stored as a new grounded case in the pgvector store. The next time a similar transaction arrives, the agent has a source to cite.

## Alternatives Considered

**LOW_CONFIDENCE tier (soft escalation):** The agent returns a verdict with a mid-range risk score (0.4–0.6) and a `LOW_CONFIDENCE` flag when it cannot cite a source. Rejected because it gives the analyst a baseless suggestion dressed as a verdict. An analyst under time pressure may confirm it without scrutiny. Dishonest confidence is more dangerous than an honest escalation.

**Configurable grounding threshold:** Allow operators to set a minimum citation requirement (e.g., "at least 2 sources") or disable enforcement entirely. Rejected for v1 — adds complexity before there is data to justify it. The eval suite will show the escalation rate on the golden dataset; if it is too high, a threshold can be introduced in v2 with evidence behind the decision.

**Trust the model's self-reported confidence:** Let the model decide when it has enough evidence. Rejected because model self-assessment of grounding is unreliable — models are known to hallucinate citations or overstate confidence. Enforcement must happen outside the model, at the application layer.

## Consequences

- Every APPROVE or DECLINE verdict is traceable to a specific rule or case — analysts can audit any decision.
- The escalation rate starts high and decreases naturally as the pgvector store grows from analyst feedback — the system becomes more capable precisely where it previously gave up.
- Implementation requires a Pydantic validator in `tessera-agent` that rejects any verdict with `decision != ESCALATE` and `cited_sources == []`, overriding the model's output before it reaches the caller.
- The system prompt must explicitly instruct the model that general training knowledge about fraud is not an acceptable citation source.
