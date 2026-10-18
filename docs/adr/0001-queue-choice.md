# ADR 0001: Queue choice for telemetry and background work

- Status: Accepted on 2026-10-17, approved by the director
- Date: 2026-10-17
- Related: docs/spec.md sections "Ingest targets" and "Engineering standards"

## Decision in one line

We use **Redis Streams** as the work queue. The queue has a hard size limit. When it is full the API
says "busy, try again in N seconds" (HTTP 429 with a `Retry-After` header) instead of falling over.
Items that fail repeatedly go to a separate "dead-letter" stream so people can inspect them. Business
events (for example "reservation approved") are first written to the database in the same transaction
as the change itself (the "transactional outbox"), then copied to the queue, so an event is never lost
or sent for a change that did not happen.

Plain-English terms:
- **Queue / stream**: a waiting line of work items. The API puts items in, workers take them out.
- **Consumer group**: a team of workers sharing one waiting line; each item goes to exactly one worker.
- **Dead-letter queue (DLQ)**: a "problem pile" for items that failed too many times.
- **Transactional outbox**: write "send this event" into the database together with the change, and a
  separate process sends it afterwards. Either both happen or neither does.

## Context

- Sites send telemetry (power in kW, rack units used). Target: **2,000 points/second sustained**,
  with a queue cap of **50,000 points** (about 25 seconds of buffer at full rate).
- Other background work: writing telemetry to Postgres, rolling it up hourly and daily, embedding
  policy documents, publishing reservation events, nightly forecasts.
- CLAUDE.md requires: bounded queues and bounded worker concurrency, 429 + Retry-After when full,
  retries with backoff and jitter, a dead-letter queue, a transactional outbox, idempotency keys,
  and tracing from API through queue to worker.
- The stack already includes Redis and Postgres. Everything must run on a laptop under Colima, on
  both arm64 and amd64.

## Options considered

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **A. Redis Streams + consumer groups** (chosen) | Append-only log inside Redis that we already run | Already in stack (no new service); about 100,000+ appends/second on a laptop, far above 2,000/s; consumer groups give one-worker-per-item, acknowledgements and redelivery of stuck items (`XAUTOCLAIM`); exact length check is cheap (`XLEN`) so the cap is easy to enforce atomically with a small Lua script | Redis keeps data in memory; with `appendfsync everysec` a Redis crash can lose up to about 1 second of accepted telemetry; we must write our own DLQ and retry counting (small) |
| B. Postgres as queue (`SELECT ... FOR UPDATE SKIP LOCKED`) | Jobs table in Postgres | Fully durable; transactional with business data; no extra moving parts | 2,000 inserts + 2,000 deletes per second on the same database that serves the API would compete with the read/write latency targets; table bloat and vacuum pressure; we still use this pattern for the **outbox** where volume is low |
| C. Celery or RQ on Redis | Python task frameworks | Familiar; retries built in | Built for "run this function" tasks, not high-rate data streams; no native hard cap on queue length (we would bolt one on); harder to trace end to end; extra dependency |
| D. Kafka (or Redpanda) | Distributed log | Highest durability and throughput; replay | A heavy new service (memory, ops, a new host port), overkill for 2,000 points/second, slower local start-up, more for the team to run |

## Decision

1. **Streams**: `cp:telemetry` (telemetry points), `cp:events` (business events from the outbox),
   `cp:documents` (document embedding jobs). Each has one consumer group.
2. **Bounded**: the stream holds only in-flight items. Workers acknowledge and then delete an item
   (`XACK` + `XDEL`), so stream length equals "items not yet done". A Lua script checks
   `XLEN + batch size <= cap` and appends the whole batch atomically, or rejects the whole batch.
   Caps: `cp:telemetry` = **50,000 points**; `cp:events` = **10,000 events**; `cp:documents` =
   **500 jobs**.
3. **When full**: the API returns **HTTP 429** with `Retry-After` in whole seconds, computed as
   `ceil(current depth / measured drain rate per second)` and clamped to **1 to 30 seconds**. If Redis
   is unreachable the API returns **HTTP 503** with `Retry-After: 5`.
4. **One point = one stream entry**, written in a single pipelined Lua call per batch (batches of up
   to **500 points**). This keeps the cap in the same unit as the target (points).
5. **Worker concurrency is bounded**: default **4 consumers per worker process**, each reading at most
   **500 entries** per call, configured by environment variable.
6. **Retries and dead-letter**: an item is retried with exponential backoff and jitter (base
   **0.5 s**, factor 2, max **30 s**) up to **5 deliveries**. After that it moves to
   `cp:dlq:<stream>` with the error, attempt count and trace id. DLQ streams are capped at
   **100,000 entries** (oldest trimmed, with a warning log). A CLI command re-queues DLQ items after a fix.
   Stuck items (worker crashed) are reclaimed after **60 s** idle with `XAUTOCLAIM`.
7. **Exactly-once effect**: delivery is "at least once"; storage is made idempotent. Telemetry rows have
   a unique key `(tenant_id, rack_id, metric, ts)` and are written with `ON CONFLICT DO NOTHING`.
   Event consumers record processed event ids in a `processed_events` table and skip repeats.
   Request retry keys: an in-progress key expires after the endpoint's hard timeout plus **60 s**. Endpoints
   with no stated timeout use a default hard timeout of **10 s** (director decision), so **70 s** (spec section 3).
8. **Transactional outbox**: domain changes and their events are written in one Postgres transaction
   to `outbox_events`. A relay process reads unpublished rows with `FOR UPDATE SKIP LOCKED`
   (batches of **100**), appends them to `cp:events`, then marks them published.
9. **Tracing**: every stream entry carries the W3C `traceparent` field so traces connect
   API -> queue -> worker -> LLM.
10. **Redis settings**: `appendonly yes`, `appendfsync everysec`, `maxmemory-policy noeviction`
    (Redis must never silently evict queue data).

## Consequences

- Good: no new infrastructure; easy to test locally with Testcontainers; the cap and 429 behaviour are
  exact and testable.
- Good: business events cannot be lost or sent for rolled-back changes (outbox).
- Cost: a Redis crash can lose up to about **1 second** of accepted telemetry. We accept this for
  telemetry (it is sampled data; the next sample replaces it). Business events are safe because they
  live in Postgres first. Listed as a risk in docs/spec.md.
- Cost: we own a small amount of retry/DLQ code (about 200 lines) instead of using a framework.
- Memory: 50,000 points of about 200 bytes each is about **10 MB** of Redis memory, well within limits.
- Revisit if sustained ingest must exceed **20,000 points/second** or multi-region replay is needed;
  Kafka/Redpanda would then be reconsidered.
