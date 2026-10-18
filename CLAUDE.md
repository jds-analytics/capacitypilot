# Project rules for all agents

## Who is who
- The director (JD) is the product owner and the only human. JD is not a developer.
- Explain every decision in plain English. No jargon without a one-line definition.
- JD approves: specs, every pull request, every deploy. You never merge, never deploy to production.

## How work flows
1. Work starts only from an approved ticket in docs/spec.md.
2. One ticket = one branch = one pull request, under ~800 changed lines.
   If it will be bigger, stop and propose a split.
3. Plan first. Show the plan, wait for approval, then build.
4. Write tests in the same pull request as the code.
5. Run `make check` and `make test` before opening a pull request. Paste the output in the PR.
6. Open the PR with the template in .github/pull_request_template.md, then STOP.

## Never
- Never commit secrets, keys, tokens or .env files.
- Never weaken, skip or delete a test to make it pass. If a test fails twice, stop and report.
- Never push to main. Never force-push. Never edit .github/workflows unless your ticket says so.
- Never use real company or customer data. Synthetic data only.
- Never add a new dependency without saying why in the PR.

## Engineering standards (every project)
- Backpressure: bounded queues and worker concurrency; API returns 429 + Retry-After when full.
- Retries: exponential backoff with jitter, max attempts, timeouts on every outbound call, idempotency keys.
- Partial failure: dead-letter queue, transactional outbox, per-item status on batch endpoints.
- Pagination: cursor/keyset only, stable sort with unique tiebreaker, max page size.
- Migrations: expand -> migrate -> contract; every migration reversible; backfills as batched jobs.
- Tenancy: tenant_id on every tenant-owned row, enforced in every query.
- Observability: OpenTelemetry traces API -> queue -> worker -> LLM; JSON logs with trace_id.
- Security: follow docs/threat-model.md; every endpoint has an authorization test; deny by default;
  containers run as non-root; GitHub Actions pinned to commit SHAs.
Each standard needs a named test that proves it.

## AI features
- Every AI feature needs: a golden dataset in evals/golden/, numeric targets in docs/spec.md,
  and `make eval` that reports quality, p95 latency and cost per request in dollars.
- Agent loops: max 8 tool calls, hard timeout, max_tokens always set.

## Stack
- TypeScript: pnpm + Turborepo, Next.js, Vitest/Playwright.
- Python: uv, FastAPI, Pydantic v2, SQLAlchemy 2 + Alembic, pytest, Testcontainers.
- Data: Postgres 16 + pgvector, Redis. Contracts: OpenAPI 3.1.
- Containers run on Colima, not Docker Desktop. Use `docker compose` (with a space).
  Images must build for linux/arm64 and linux/amd64.

## Commands
- make up      start everything locally (Docker)
- make check   lint + type-check
- make test    all tests
- make eval    AI evaluations
- make down    stop everything

## Lessons learned
See docs/lessons.md. Read it before starting any ticket.