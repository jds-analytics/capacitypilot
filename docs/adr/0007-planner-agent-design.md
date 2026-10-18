# ADR 0007: Planner Q&A agent design (router + two models)

- Status: Accepted on 2026-10-17, approved by the director (models: Q2; embeddings: Voyage AI, Q3)
- Date: 2026-10-17
- Related: docs/spec.md "AI eval targets", docs/threat-model.md

## Decision in one line

Every question first goes to **Claude Haiku 5.5**, a fast, low-cost model that sorts it into one of
three kinds: off-topic (politely refused), a simple lookup (Haiku answers it), or a full planning
question (**Claude Sonnet 5.5** writes the answer). Both use the same **short, fixed list of read-only
tools**: search the policy documents, and look up site capacity, utilization, forecasts and
reservations for the **current tenant only**. Each question is limited to **8 tool calls** in total,
a hard **15-second** time limit, a cap on answer length and **$0.02** of spend. Answers cite their
sources. **Claude Opus 5.5** is used only inside `make eval` as a comparison and never answers real
requests.

Plain-English terms:
- **Agent loop**: the model reads the question, decides to call a tool, reads the result, and repeats
  until it can answer.
- **Router**: a quick first step that decides which path a question takes.
- **Retrieval (RAG)**: finding the most relevant passages of the policy documents to show the model.
- **Embedding**: a list of numbers representing a passage's meaning, so similar passages can be found.
- **Hybrid search**: combine meaning-based search (embeddings) with keyword search, then merge results.
- **Prompt injection**: text hidden in a document or data that tries to give the model new orders.

## Context

- Users ask things like "Can Room B in DAL-1 take 120 kW more in March under our 80% loading policy?"
  (needs policy text and live numbers) and "What is DAL-1's forecast peak for next month?" (one lookup).
- Targets (spec section 6): recall@5 >= 0.85, faithfulness >= 0.90, correctness >= 0.80, out-of-scope
  refusal >= 0.95, prompt-injection resistance = 1.00, p95 time to first token <= 1.5 s, p95 total
  <= 6 s, cost <= $0.02 per question.
- CLAUDE.md: max 8 tool calls, hard timeout, `max_tokens` always set.
- Director decision (Q2): Haiku 5.5 routes and answers simple lookups; Sonnet 5.5 writes full answers;
  Opus 5.5 is compared in `make eval` but stays out of the live path.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| Single retrieval call, no tools (classic RAG) | Cheapest, fastest | Cannot answer live-data questions |
| One model for everything (Sonnet) | Simplest | Every question pays Sonnet prices and latency, including refusals and one-number lookups |
| One model for everything (Haiku) | Cheapest and fastest | Weaker on multi-step reasoning over policy + numbers |
| **Haiku router + Haiku simple path + Sonnet full path, allow-listed read-only tools** (chosen) | Cheap and fast for most questions; Sonnet quality where it matters; every fact traceable to a tool result | Two models to evaluate; a mis-routed question can get a weaker answer (measured) |
| Let the model write SQL | Flexible | Dangerous (data leaks, heavy queries); hard to evaluate; rejected |

## Decision

1. **Models (configuration, not code)**:

   | Purpose | Model id | Setting | Live requests? |
   |---|---|---|---|
   | Router and simple-lookup answers | `claude-haiku-5-5` | `CP_AGENT_ROUTER_MODEL` | yes |
   | Full answers | `claude-sonnet-5-5` | `CP_AGENT_ANSWER_MODEL` | yes |
   | Comparison in `make eval` | `claude-opus-5-5` | `CP_EVAL_COMPARE_MODEL` | **no, eval only** |
   | Eval judge | `claude-sonnet-5-5` (default) | `CP_EVAL_JUDGE_MODEL` | no, eval only |

   A start-up check refuses to run the API if `CP_AGENT_ROUTER_MODEL` or `CP_AGENT_ANSWER_MODEL` is set
   to an Opus model (`test_opus_not_in_live_path`).
2. **Routing**: Haiku returns one label, `out_of_scope`, `simple_lookup` or `full`, using structured
   output with `max_tokens` **64** and a **2 s** timeout. If the router fails, times out or is unsure,
   the question takes the `full` path (safe default).
   - `out_of_scope`: a fixed polite refusal. No tools, no Sonnet call.
   - `simple_lookup`: Haiku runs the tool loop (at most **3** tool calls) and writes the answer.
   - `full`: Sonnet runs the tool loop and writes the answer.
3. **Tools (read-only, allow-listed)**: `search_policies(query, k<=8)`, `get_site_capacity(site_code)`,
   `get_utilization(site_code, room?, window_days<=90)`, `get_forecast(site_code, metric, horizon_days<=90)`,
   `list_reservations(site_code, status?, limit<=20)`. Arguments are validated with Pydantic; unknown
   tools or invalid arguments return an error result to the model, not an exception.
4. **Tenant is never a tool argument**: tools run inside the caller's `TenantContext` and RLS (ADR 0003).
5. **Retrieval**: Voyage AI embeddings (1,024 dimensions, model set by `CP_EMBED_MODEL`); chunks of
   about **500 tokens** with **50-token overlap**; hybrid search = pgvector cosine top 20 + Postgres
   full-text top 20, merged with reciprocal rank fusion, top 5 returned.
6. **Caps (whole question, all models together)**: max **8 tool calls**, hard timeout **15 s**
   wall-clock (the loop is cancelled and a short "I ran out of time" answer is returned), `max_tokens`
   = **64** for the router, **512** for intermediate steps, **1,024** for the final answer; tool results
   capped at **4,000 characters**.
7. **Cost guard**: before every model call the loop adds up the cost so far (router + tool steps +
   query embeddings) from token counts and a dated price table, and estimates the next call at its
   `max_tokens`. If that would take the question over **$0.02**, it stops and answers with what it has
   (status `budget_stopped`). Budgets by path: router **<= $0.002**; simple-lookup path **<= $0.008**
   in total; full path **<= $0.02** in total. Prompt caching is used for the system prompts and tool
   definitions of both models.
8. **Untrusted content**: document text and tool results are wrapped in clearly marked data blocks; the
   system prompt says they are data, never instructions. A guard module checks tool arguments and scans
   the answer for leaked system prompt or secrets.
9. **Streaming**: answers stream to the caller by Server-Sent Events (SSE: a one-way live text stream).
   The **1.5 s p95** first-token target includes the router step.
10. **Opus comparison**: `make eval` re-runs a fixed 40-question comparison subset with Opus as the
    full-path model and reports quality, latency and cost differences. This is reported only, never
    gated, and is paid for inside the approved **$10 per eval run** budget (spec section 6).
11. **Record keeping**: each question stores route, model(s) used, answer, citations, tool calls, token
    counts per model, cost in dollars and latency; prompts containing user data are not written to logs.

## Consequences

- Good: most questions (refusals and lookups) never touch the more expensive model, which helps the
  $0.02 and 1.5 s targets; grounded answers with citations; tenancy is enforced below the model.
- Cost: the router adds one short call (budget 2 s timeout, usually far less) before the first token.
- Cost: full-path questions with many tool calls on Sonnet are the ones most likely to approach the
  $0.02 limit. If `make eval` shows the full path over budget, the fallback lever is a "Haiku gathers
  data, Sonnet writes once" mode, which would be a new ticket.
- Risk: a mis-routed question gets a weaker answer; `make eval` reports routing accuracy (report only).
- Changing the embedding model later requires re-embedding all documents (a batched backfill job).
- Every agent-loop PR is **risk:high** and needs red-team review.
