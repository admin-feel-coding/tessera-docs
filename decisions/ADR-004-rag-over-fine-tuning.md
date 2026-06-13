# ADR-004: RAG-Based Feedback Loop over Fine-Tuning

## Status
Accepted

## Context
As the system accumulates analyst corrections, there are two ways to incorporate that knowledge into future decisions: fine-tuning the underlying model on the corrected cases, or storing the cases in a vector database and retrieving them at inference time (RAG). The choice affects auditability, update latency, operational complexity, and cost — all of which are critical constraints for a fraud decisioning system.

## Decision
Use RAG (Retrieval Augmented Generation) via pgvector as the sole mechanism for incorporating analyst feedback into future verdicts. Fine-tuning is explicitly out of scope for v1 and is not the intended direction for v2.

When an analyst corrects a verdict, the corrected transaction — along with the analyst's decision, note, and decisive signals — is embedded and stored as a new case in the pgvector store. The next time a similar transaction arrives, the agent retrieves this case via similarity search and uses it as cited evidence. The model itself does not change.

## Why Not Fine-Tuning

**1. Claude does not support fine-tuning.**
Anthropic does not offer public fine-tuning for Claude models. Pursuing fine-tuning would require switching to a different model provider, which is a significant architectural change with no clear benefit over RAG for this use case.

**2. Fine-tuning destroys auditability.**
When a model is fine-tuned, knowledge is baked into its weights. A verdict produced by a fine-tuned model cannot be traced back to a specific historical case or rule — the reasoning is opaque. Tessera's core design principle is that every verdict must cite a grounded source. RAG satisfies this requirement structurally: the retrieved cases appear explicitly in the agent's context and are cited in the verdict. Fine-tuning makes this impossible.

**3. Fraud patterns change faster than fine-tuning cycles.**
Fine-tuning requires collecting labeled data, running training jobs, evaluating the new model, and redeploying — a cycle measured in weeks. Fraud patterns evolve continuously. With RAG, an analyst correction made today is available as a retrievable case within the same hour. The feedback loop is operational, not a batch process.

**4. Operational cost and complexity.**
Fine-tuning requires GPU infrastructure, labeled dataset pipelines, and model evaluation before every deployment. RAG requires a pgvector insert and an embedding API call — negligible cost and no additional infrastructure.

## How RAG Improves Over Time

The system starts with a seeded synthetic dataset of ~100 cases. Every escalation that a human analyst resolves becomes a new case in the store. Over time, the pgvector store grows denser in the regions of transaction space where the agent previously lacked evidence — meaning the agent's escalation rate decreases naturally as it accumulates real, grounded cases. The system becomes most capable precisely where it was previously weakest.

## Alternatives Considered

**Fine-tuning on corrected cases:** Rejected for the three reasons above — Claude does not support it, it eliminates auditability, and the feedback cycle is too slow for a dynamic fraud environment.

**In-context learning only (no retrieval):** Putting all known cases directly in the system prompt. Rejected because the context window cannot hold more than a few dozen cases, and the set grows unboundedly as analysts submit corrections.

**Hybrid RAG + fine-tuning:** Retrieving cases at inference time while also periodically fine-tuning on the accumulated dataset. Rejected for v1 — adds significant complexity without a clear improvement over RAG alone when auditability is a hard requirement. Revisit if the eval suite shows RAG alone cannot reach the 80% accuracy target.

## Consequences

- Every verdict improvement is directly traceable — analysts can see which retrieved case influenced a decision.
- The pgvector store is the system's growing memory. Its quality and coverage are the primary levers for improving agent accuracy over time.
- Embedding quality matters: the similarity search is only as good as the embeddings. The embedding model choice should be revisited if retrieval quality becomes a bottleneck.
- No GPU infrastructure required. The feedback loop runs entirely on the existing PostgreSQL instance.
