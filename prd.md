# Tessera — AI Fraud Analyst Agent

*Product Requirements Document (v1)*

**Author:** Juan Sarabia
**Status:** Draft
**Last updated:** June 2026

> **On the name:** In ancient Rome, a *tessera* was a token used as proof, identity, or access. The name reflects the product's core principle — every verdict is a grounded piece of evidence — and nods to the tokenization world of payments it serves.

---

## 1. The Problem

Fraud teams at e-commerce and fintech companies rely on rule engines to approve or decline most transactions in milliseconds. But rules can't resolve every case — a meaningful slice of traffic falls into an **ambiguous tier**: transactions that aren't clearly fraud and aren't clearly legitimate.

Today, these ambiguous cases land in a **manual review queue**. Human fraud analysts investigate each one — checking the user's history, device, IP, and similar past cases — and decide. This is slow (several minutes per case), expensive (skilled analysts don't scale), and inconsistent (decisions vary by analyst). As transaction volume grows, the review queue grows with it, and the team can't keep up.

## 2. The User

- **Primary user:** the fraud analyst team that works the manual review queue. The agent reduces their workload by handling the routine ambiguous cases automatically.
- **Secondary beneficiary:** the risk/fraud lead, who gets faster decisions, consistent reasoning, and a clear audit trail for every verdict.

For these users to trust the system, every decision must be **explainable and traceable** — they need to see *why* the agent reached a verdict, not just the verdict itself.

## 3. The Solution

An LLM-based agent that acts as an **automated fraud analyst for ambiguous transactions** — the cases rule engines can't resolve. Given a transaction, the agent investigates using live data and historical cases, then returns a structured, grounded verdict (risk score, decision, reasoning, cited evidence).

**How it works (three knowledge layers):**
- **System prompt** — stable, owner-defined business rules.
- **Tools** — live data queries (user purchase history, IP/device fingerprint, internal blacklists).
- **RAG (pgvector)** — retrieval of the most similar historical cases.

The agent reasons in a loop — calling tools, gathering evidence — and returns a JSON verdict validated against a strict schema.

## 4. What This Is NOT (Scope Boundaries)

- **Not a replacement for the rule engine.** Rules still handle the bulk of traffic with millisecond decisions. The agent only touches the ambiguous tier (~10% of volume).
- **Not real-time decisioning for all transactions.** It targets the analyst-review queue, not the hot path.
- **Not autonomous for high-risk cases.** If the agent cannot ground a verdict in a cited rule or retrieved case, it **rejects the verdict and escalates to a human analyst.**
- **Not powered by the LLM's general knowledge.** The agent is explicitly forbidden from using general training knowledge about fraud. Every verdict must cite a source (an owner-defined rule or a retrieved case) — otherwise it escalates.

## 5. Definition of Success

Given a golden dataset of 50–100 ambiguous cases (each with a known "correct" human decision), v1 is successful when the agent:

- Matches the human analyst decision **≥ 80%** of the time.
- Returns a verdict in **under 5 seconds** per case.
- Costs **under $0.10** per decision.
- **Cites a grounded source** on every verdict, or escalates when it can't.

These metrics are produced by an automated **eval suite** that runs the full golden dataset before any prompt change.

## 6. Feedback Loop (Human-in-the-Loop)

When an analyst reviews a verdict, they can correct the decision, add a short note explaining the reasoning, and confirm which signals were decisive.

**Low-friction by design:** the analyst confirms or edits what's already there rather than writing from scratch.

**How feedback feeds back:**
- The corrected transaction + human decision + note + decisive signals is stored as a **new grounded case in the pgvector store.**
- The next time a similar transaction arrives, the agent retrieves this case via RAG and uses it as cited evidence.
- The system improves by **growing its case memory**, not by retraining the model.

## 7. Production Discipline

- **Observability:** Langfuse traces every agent run — tool calls, reasoning, token usage, cost.
- **Guardrails:** timeouts, max tool-calls per run, cost ceilings per request, fallback to rule-based decisioning if the LLM fails.
- **Grounding enforcement:** uncited verdicts are rejected and escalated, by design.

## 8. Tech Stack

Python (FastAPI) · Go (net/http) · Next.js App Router · Anthropic Claude API · PostgreSQL + pgvector · Langfuse Cloud · GCP

## 9. Build Sequence (7 weeks)

| Week | Milestone |
|---|---|
| 1 | Working agent: transaction → JSON verdict |
| 2 | Langfuse observability |
| 3 | Real tools: data queries in a loop |
| 4 | Synthetic data + RAG (pgvector) |
| 5 | Eval suite (the proof) |
| 6 | Guardrails + grounding enforcement |
| 7 | Human-in-the-loop feedback (analyst corrections → new RAG cases) |
| 8 | Polish, README |
