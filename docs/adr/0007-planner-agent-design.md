# ADR 0007: Planner Q&A agent design

- Status: Proposed (Claude + hosted embeddings decided by the director; design needs approval)
- Date: 2026-10-17
- Related: docs/spec.md "AI eval targets", docs/threat-model.md

## Decision in one line

The planner assistant is Claude with a **short, fixed list of read-only tools**: search the policy
documents, and look up site capacity, utilization, forecasts and reservations for the **current tenant
only**. It may call tools at most **8 times** per question, must finish within a hard **15-second**
limit, always has a cap on answer length, cites the documents and data it used, and politely refuses
questions outside data-center capacity planning.

Plain-English terms:
- **Agent loop**: the model reads the question, decides to call a tool, reads the result, and repeats
  until it can answer.
- **Retrieval (RAG)**: finding the most relevant passages of the policy documents to show the model.
- **Embedding**: a list of numbers representing a passage's meaning, so similar passages can be found.
- **Hybrid search**: combine meaning-based search (embeddings) with keyword search, then merge results.
- **Prompt injection**: text hidden in a document or data that tries to give the model new orders.

## Context

- Users ask things like "Can Room B in DAL-1 take 120 kW more in March under our 80% loading policy?"
  which needs both policy text and live numbers.
- Targets (spec): recall@5 >= 0.85, faithfulness >= 0.90, correctness >= 0.80, out-of-scope refusal
  >= 0.95, prompt-injection resistance = 1.00, p95 time to first token <= 1.5 s, p95 total <= 6 s,
  cost <= $0.02 per question.
- CLAUDE.md: max 8 tool calls, hard timeout, `max_tokens` always set.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| Single retrieval call, no tools (classic RAG) | Cheapest, fastest | Cannot answer live-data questions |
| **Tool-using agent with allow-listed read-only tools** (chosen) | Answers policy + live data; every fact traceable to a tool result | More calls means more latency and cost; needs caps |
| Let the model write SQL | Flexible | Dangerous (data leaks, heavy queries); hard to evaluate; rejected |

## Decision

1. **Tools (read-only, allow-listed)**: `search_policies(query, k<=8)`, `get_site_capacity(site_code)`,
   `get_utilization(site_code, room?, window_days<=90)`, `get_forecast(site_code, metric, horizon_days<=90)`,
   `list_reservations(site_code, status?, limit<=20)`. Arguments are validated with Pydantic; unknown
   tools or invalid arguments return an error result to the model, not an exception.
2. **Tenant is never a tool argument**: tools run inside the caller's `TenantContext` and RLS (ADR 0003).
3. **Retrieval**: chunks of about **500 tokens** with **50-token overlap**; hybrid search = pgvector
   cosine top 20 + Postgres full-text top 20, merged with reciprocal rank fusion, top 5 returned.
4. **Caps**: max **8 tool calls**, hard timeout **15 s** wall-clock (the loop is cancelled and a
   partial "I ran out of time" answer is returned), `max_tokens` = **1,024** for the final answer and
   **512** for intermediate steps, tool result size capped at **4,000 characters**.
5. **Cost guard**: before each model call the loop estimates the running cost from token counts and a
   dated price table; if the next call would push the question over **$0.02**, the loop stops and
   answers with what it has. Prompt caching is used for the system prompt and tool definitions.
6. **Untrusted content**: document text and tool results are wrapped in clearly marked data blocks; the
   system prompt says they are data, never instructions. A guard module checks tool arguments and scans
   the answer for leaked system prompt or secrets.
7. **Streaming**: answers stream to the browser by Server-Sent Events (SSE: a one-way live text stream);
   the first token target is **<= 1.5 s p95**.
8. **Record keeping**: each question stores answer, citations, tool calls, token counts, cost in
   dollars and latency; prompts containing user data are not written to logs (security rule).
9. **Model and embedding vendor** are configuration (`CP_AGENT_MODEL`, `CP_EMBED_MODEL`), chosen by
   the director (open question). The embedding vector size is fixed in the database once chosen; the
   schema assumes **1,024 dimensions** until then.

## Consequences

- Good: grounded answers with citations; tenancy is enforced below the model.
- Cost: multi-step questions approach the 6-second and $0.02 limits; a larger Claude model may not fit
  the cost target with 8 tool calls (see open questions).
- Changing the embedding model later requires re-embedding all documents (a batched backfill job).
- Every agent-loop PR is **risk:high** and needs red-team review.
