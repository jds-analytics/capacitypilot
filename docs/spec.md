# CapacityPilot specification

- Status: **Draft for director approval** (nothing is built until JD approves this spec)
- Author: architect agent
- Date: 2026-10-17
- Decision records: docs/adr/0001 to 0008

How to read this document: every section starts with **Decision** in plain English. The detail below
it is for the agents building the system. Words in **bold** with a dash after them are defined on first
use. Every target is a number with a unit. Anything marked **(PROPOSED)** is my recommendation and
needs your yes/no.

---

## 1. Problem and users

**Decision:** CapacityPilot helps data-center planners answer one question with confidence: *"Do we have
room for this, and will we still have room in 3 months?"* It does four things: collects power and
space readings from sites, forecasts each site 90 days ahead every night, runs a request -> approve ->
allocate workflow for reserving capacity, and offers an AI assistant that answers planning questions
using the company's policy documents and live numbers.

### The problem today (assumed)
- Capacity data lives in spreadsheets and monitoring tools that do not talk to each other.
- Reservations are made by email; two teams can be promised the same power.
- Forecasts are done by hand, rarely, and without a range of uncertainty.
- Planning policies (loading limits, approval rules) live in documents few people read.

### Users and what they need

| User | Role in the system | What they do |
|---|---|---|
| Capacity planner | `viewer`, `requester` | Checks headroom, reads forecasts, asks the assistant, requests capacity |
| Capacity approver (site or regional lead) | `approver` | Approves or rejects requests, allocates racks, releases capacity |
| Platform admin | `admin` | Manages sites, rooms, racks, policy documents; triggers forecast runs |
| Telemetry sender (a machine, not a person) | `ingestor` | Sends power and rack-unit readings |
| Director (JD) | product owner | Approves specs, PRs and deploys; watches quality reports |

### Out of scope for this version
Billing, cooling/thermal modelling, network ports, real customer data, mobile apps, multi-region
deployment, email/SMS notifications (events are recorded so they can be added later).

### Synthetic world we build and test against
All data is synthetic, generated from a fixed random seed (`SEED=42`) so every run is identical.

| Thing | Amount |
|---|---|
| Tenants (customer organisations) | 3 |
| Sites | 12 (4 per tenant) |
| Rooms | 4 to 8 per site (about 72 total) |
| Racks | 20 to 60 per room (about 2,500 total), 42 RU each |
| History | 730 days of daily site/room rollups; 2 days of raw readings at 60-second intervals |
| Normal telemetry rate | about 85 points/second (2,500 racks x 2 metrics / 60 s); the 2,000 points/second target is about 24x headroom |
| Data quirks | weekly and yearly seasonality, growth trend, step changes when a customer moves in, 1% gaps, 0.5% bad rows |

---

## 2. Domain model

**Decision:** A **tenant** (customer) owns **sites**. A site contains **rooms**; a room contains
**racks**. Racks send **readings** of power (kW) and rack units used (RU). Every night we produce a
**forecast** per site for kW and RU. People create **reservations** asking for kW and RU in a room or
a specific rack; an approver approves it and then **allocates** it to actual racks. Policy
**documents** are split into passages that the AI assistant can search. Every customer's data is kept
apart by a tenant id on every row (ADR 0003).

### In plain English

- A **tenant** is one customer organisation. Nothing crosses between tenants.
- A **site** is a data-center building (for example "DAL-1"). It has a total power budget in kW and a
  time zone.
- A **room** is a data hall inside a site. It has its own power budget in kW. Its space is the sum of
  its racks' space.
- A **rack** is a cabinet in a room. It has a space capacity (usually 42 RU, where one **RU** = one
  1.75-inch slot) and a power capacity in kW.
- A **reading** (telemetry point) is one measurement: "rack R at time T used 7.4 kW" or "had 31 RU
  in use". Readings are rolled up into **hourly** and **daily** summaries.
- **Capacity** = what a room or rack can hold. **Committed** = what is already allocated plus what is
  approved but not yet allocated. **Used** = what readings show is actually drawn. **Headroom** =
  capacity minus the larger of committed and used. Exact formulas live in docs/metrics.md.
- A **reservation** is a request for kW and RU, at room level ("anywhere in Room B") or rack level
  ("rack B-14"), from a start date to an optional end date. It moves through these states:

```
requested --approve--> approved --allocate--> allocated --release--> released
    |                     |   \
    |--reject--> rejected |    \--(not allocated within 14 days, PROPOSED)--> expired
    |--cancel--> cancelled <--cancel--|
```

  Approval and allocation both check headroom. Two approvals racing for the last 10 kW cannot both
  succeed (the room row is locked during the check).
- An **allocation** ties part of a reservation to one specific rack. A room-level reservation may be
  split across several racks.
- A **forecast run** is one nightly job. It produces one **forecast** per site and metric, made of 90
  daily **forecast points**, each with a best guess and 80% and 95% ranges.
- A **policy document** is an uploaded Markdown or text file (for example "Power loading policy").
  It is split into **chunks** (passages of about 500 tokens) with an **embedding** (a list of numbers
  representing meaning) for search.
- A **planner question** records each question to the assistant, its answer, citations, tools used,
  time taken and cost.
- Supporting records: **outbox events** (events waiting to be sent, ADR 0001), **processed events**
  (to ignore repeats), **idempotency keys** (so a retried request does not happen twice),
  **audit log** (who changed what), **job checkpoints** (so batch jobs can resume).

### Precise entity list

Conventions: ids are UUIDv7 (time-ordered) generated by the app; all times are UTC `timestamptz`;
every table marked **[T]** is tenant-owned: it has `tenant_id uuid NOT NULL`, forced row-level
security, indexes leading on `tenant_id`, and composite foreign keys `(tenant_id, parent_id)`.

| Entity (table) | Key fields | Relationships and rules |
|---|---|---|
| `tenants` | id, name, slug (unique), created_at | Not tenant-owned. Readable only by `cp_scheduler` and admin tooling |
| `sites` [T] | id, code (unique per tenant), name, region, timezone, capacity_kw (>0), created_at, updated_at, archived_at | Has many rooms |
| `rooms` [T] | id, site_id, name (unique per site), capacity_kw (>0), created_at, updated_at, archived_at | Belongs to site; has many racks. Sum of room capacity_kw may not exceed site capacity_kw (checked on write) |
| `racks` [T] | id, room_id, site_id (denormalised for queries), label (unique per room), capacity_ru (1..60, default 42), capacity_kw (>0), created_at, updated_at, archived_at | Belongs to room |
| `telemetry_points` [T] | tenant_id, site_id, room_id, rack_id, metric (`power_kw` or `ru_used`), ts, value, ingested_at, batch_id | Partitioned by day on `ts`. Unique `(tenant_id, rack_id, metric, ts)`. `power_kw` 0..rack capacity_kw x 1.2; `ru_used` 0..capacity_ru |
| `telemetry_hourly` [T] | tenant_id, site_id, room_id, metric, hour, avg, max, min, sample_count | Unique `(tenant_id, room_id, metric, hour)`; upserted by rollup job |
| `telemetry_daily` [T] | tenant_id, site_id, room_id, metric, day, avg, max, min, sample_count | Unique `(tenant_id, room_id, metric, day)`; forecast input |
| `forecast_runs` [T] | id, started_at, finished_at, status (`running`, `succeeded`, `partial`, `failed`), model_version, trigger (`schedule` or `manual`), sites_total, sites_failed | Has many forecasts |
| `forecasts` [T] | id, run_id, site_id, metric (`peak_kw` or `ru_used`), status (`ok`, `insufficient_history`, `failed`), model (`ets`, `mstl_ets`, `seasonal_naive`), history_days, backtest_wape, created_at | Has 90 forecast points. Latest per (site, metric) is what the API serves |
| `forecast_points` [T] | forecast_id, day, yhat, lo80, hi80, lo95, hi95 | Unique `(tenant_id, forecast_id, day)` |
| `reservations` [T] | id, site_id, room_id, rack_id (nullable), requested_kw (>0), requested_ru (>=0), start_date, end_date (nullable, >= start_date), status, requested_by, justification (<=2,000 chars), decided_by, decided_at, decision_note, expires_at, created_at, updated_at | Room-level if rack_id is null. Status transitions only as in the diagram |
| `allocations` [T] | id, reservation_id, rack_id, kw, ru, allocated_at, released_at | Belongs to reservation and rack. Active = released_at is null |
| `reservation_events` [T] | id, reservation_id, from_status, to_status, actor_id, note, at, trace_id | Append-only history, written in the same transaction as the change |
| `policy_documents` [T] | id, title, filename, content_type (`text/markdown` or `text/plain`), size_bytes (<= 2 MB), content_sha256, status (`pending`, `embedded`, `failed`), chunk_count, uploaded_by, created_at, deleted_at | Unique `(tenant_id, content_sha256)` (same file twice is one document) |
| `document_chunks` [T] | id, document_id, ordinal, text, token_count, tsv (full-text vector), embedding `vector(1024)` | HNSW cosine index on embedding; GIN index on tsv. Dimension changes if the chosen embedding model differs (open question Q3) |
| `planner_questions` [T] | id, actor_id, question (<= 2,000 chars), answer, status (`answered`, `refused`, `timed_out`, `budget_stopped`, `error`), citations (json), tool_calls (json), input_tokens, output_tokens, cost_usd, ttft_ms, total_ms, model, created_at | One row per question |
| `outbox_events` [T] | id, stream, event_type, aggregate_type, aggregate_id, payload (json), traceparent, created_at, published_at, attempts | Written in the same transaction as the business change |
| `processed_events` [T] | event_id, consumer, processed_at | Unique `(consumer, event_id)` so repeats are skipped |
| `idempotency_keys` [T] | key, method, path, request_sha256, status_code, response_body, created_at, expires_at (24 h) | Primary key `(tenant_id, key)` |
| `audit_log` [T] | id, actor_id, action, target_type, target_id, at, trace_id, detail (json, no secrets) | Append-only |
| `job_checkpoints` | job_name, tenant_id (nullable), cursor (json), updated_at | Resumable batch jobs (ADR 0004) |

Users are **not** stored yet: `actor_id` is the user id string supplied by the auth layer (ADR 0008).
A `users`/`memberships` table is added when the director chooses the auth provider (CP-44).

### Roles and permissions (PLACEHOLDER until auth is chosen, open question Q1 and Q5)

| Action | viewer | requester | approver | admin | ingestor |
|---|---|---|---|---|---|
| Read sites, rooms, racks, capacity, utilization, forecasts, reservations | yes | yes | yes | yes | no |
| Create, edit, archive sites, rooms, racks | no | no | no | yes | no |
| Send telemetry | no | no | no | no | yes |
| Create a reservation; cancel own reservation | no | yes | yes | yes | no |
| Approve, reject, allocate, release | no | no | yes (not own request, PROPOSED) | yes | no |
| Upload or delete policy documents | no | no | no | yes | no |
| Ask the planner assistant | yes | yes | yes | yes | no |
| Trigger a forecast run | no | no | no | yes | no |

---

## 3. API outline (OpenAPI 3.1)

**Decision:** One versioned web API under `/v1`, described by an OpenAPI 3.1 contract kept in
`contracts/openapi/` (split by resource, bundled automatically, ADR 0005). Every endpoint requires
login except two health checks. Lists use cursor pagination with at most 100 items per page (ADR 0002).
Every create or state-changing request accepts an `Idempotency-Key` so a retried click never happens
twice. Errors use one standard format.

### Conventions (apply to every endpoint)
- Base URL locally: `http://localhost:18000/v1`. Web app: `http://localhost:18080`.
- **Auth**: `Authorization: Bearer <token>`; the auth layer yields `tenant_id`, `user_id`, `roles`.
  Missing/invalid token -> **401**. Wrong role -> **403**. Another tenant's object -> **404**.
- **Idempotency**: `Idempotency-Key` header (1-128 printable characters) is **required** on every
  `POST` and accepted on `PATCH`/`DELETE`. Same key + same body within 24 h returns the original
  response; same key + different body -> **422**; same key still in progress -> **409**.
- **Pagination**: `limit` (1-100, default 50), `cursor`, `sort` from a per-endpoint allow-list.
  Response `{items, next_cursor}`.
- **Errors**: RFC 9457 `application/problem+json` with `type`, `title`, `status`, `detail`,
  `trace_id`. No stack traces. Types: `validation`, `unauthorized`, `forbidden`, `not-found`,
  `conflict`, `capacity-exceeded`, `invalid-transition`, `invalid-cursor`, `idempotency-mismatch`,
  `queue-full`, `rate-limited`, `unavailable`, `payload-too-large`.
- **Busy signals**: **429** + `Retry-After` (seconds) when a queue is full or a rate limit is hit;
  **503** + `Retry-After` when a dependency (Redis) is down.
- **Tracing**: accepts and returns W3C `traceparent`; every response has `X-Trace-Id`.
- **Limits**: request body <= **1 MB** (documents upload <= **2 MB**); rate limit **50 requests/second
  per tenant** and **20 requests/second per IP** for unauthenticated attempts (PROPOSED).

### Endpoints

| Method and path | Purpose | Roles | SLO class |
|---|---|---|---|
| `GET /healthz` | Process is alive (public) | public | - |
| `GET /readyz` | Postgres and Redis reachable (public) | public | - |
| `GET /v1/sites` | List sites (sort: `code`, `created_at`) | read roles | read |
| `POST /v1/sites` | Create site | admin | write |
| `GET /v1/sites/{site_id}` | Get site | read roles | read |
| `PATCH /v1/sites/{site_id}` | Update site | admin | write |
| `DELETE /v1/sites/{site_id}` | Archive site (409 if active reservations) | admin | write |
| `GET /v1/sites/{site_id}/rooms` | List rooms (sort: `name`, `created_at`) | read roles | read |
| `POST /v1/sites/{site_id}/rooms` | Create room | admin | write |
| `GET, PATCH, DELETE /v1/rooms/{room_id}` | Get, update, archive room | read roles / admin | read / write |
| `GET /v1/rooms/{room_id}/racks` | List racks (sort: `label`, `created_at`) | read roles | read |
| `POST /v1/rooms/{room_id}/racks` | Create rack | admin | write |
| `GET, PATCH, DELETE /v1/racks/{rack_id}` | Get, update, archive rack | read roles / admin | read / write |
| `GET /v1/sites/{site_id}/capacity` | Per-room capacity, committed, used, headroom (kW and RU) | read roles | read |
| `GET /v1/sites/{site_id}/utilization` | Hourly or daily rollups, `metric`, `from`, `to`, `grain`; paginated (sort: `ts`) | read roles | read |
| `POST /v1/telemetry/batches` | Ingest up to 500 points; **202** with per-item status | ingestor | ingest |
| `POST /v1/reservations` | Create reservation (room or rack level) | requester, approver, admin | write |
| `GET /v1/reservations` | List; filters `site_id`, `status`, `requested_by` (sort: `created_at`, `start_date`) | read roles | read |
| `GET /v1/reservations/{id}` | Get with allocations and history | read roles | read |
| `POST /v1/reservations/{id}/approve` | Approve (headroom check; 409 `capacity-exceeded`) | approver, admin | write |
| `POST /v1/reservations/{id}/reject` | Reject with note | approver, admin | write |
| `POST /v1/reservations/{id}/allocate` | Allocate to racks (auto-pick or explicit list) | approver, admin | write |
| `POST /v1/reservations/{id}/cancel` | Cancel (own request, or approver/admin) | requester, approver, admin | write |
| `POST /v1/reservations/{id}/release` | Release allocated capacity | approver, admin | write |
| `GET /v1/sites/{site_id}/forecasts/latest?metric=peak_kw\|ru_used` | Latest 90-day forecast with intervals | read roles | read |
| `GET /v1/forecast-runs` | List runs (sort: `started_at`) | read roles | read |
| `POST /v1/forecast-runs` | Trigger a run now (**202**) | admin | write |
| `POST /v1/documents` | Upload `.md` or `.txt` (multipart, <= 2 MB); **202**, embedding happens in background | admin | write |
| `GET /v1/documents` | List (sort: `created_at`, `title`) | read roles | read |
| `GET /v1/documents/{id}` | Status and chunk count | read roles | read |
| `DELETE /v1/documents/{id}` | Delete document and its chunks | admin | write |
| `POST /v1/planner/questions` | Ask the assistant; streams `text/event-stream` (events: `meta`, `tool`, `token`, `citation`, `done`, `error`) | human roles | AI |
| `GET /v1/planner/questions` | List my past questions (sort: `created_at`) | human roles | read |
| `GET /v1/planner/questions/{id}` | Stored answer, citations, tool trace, cost | human roles | read |

### Telemetry batch shape
Request: `{"points": [{"rack_id": uuid, "metric": "power_kw"|"ru_used", "ts": RFC 3339 UTC, "value": number}]}`,
1-500 points, body <= 256 KB. `ts` may be at most **5 minutes** in the future and **7 days** in the past.
Response **202**: `{"batch_id", "accepted": n, "rejected": n, "items": [{"index", "status": "accepted"|"rejected", "reason"}]}`.
The whole batch is refused with **429** (nothing enqueued) if accepting it would exceed the queue cap.
Body that fails schema validation entirely -> **422**.

---

## 4. Service level objectives (SLOs)

**Decision:** At **50 requests per second**, 95% of reads finish within **200 ms** and 95% of writes
within **400 ms**. These are measured by an automated load test on a defined laptop setup, and a ticket
is not "done" if it breaks them.

Plain English: **p95** means "95 out of 100 requests are at least this fast". An **SLO** is a target
we promise to meet and measure.

| SLO | Target | How measured |
|---|---|---|
| Read latency (GET) | p95 **<= 200 ms** at **50 requests/second** | Load test (CP-40), 10 min steady state after 1 min warm-up, 80% reads / 20% writes over inventory, capacity, reservations and forecast endpoints |
| Write latency (POST, PATCH, DELETE) | p95 **<= 400 ms** at **50 requests/second** | Same run |
| Server error rate | **<= 0.1%** of requests return 5xx (429 excluded) | Same run |
| Planner assistant | see section 6 | `make eval` |
| Ingest | see section 5 | Ingest load test |
| Forecast freshness (PROPOSED) | Nightly run starts **02:00 UTC** and finishes within **30 min** for **500 sites x 2 metrics**; every eligible site has a forecast **<= 26 h** old | Job metrics, CP-28 test |
| Document embedding (PROPOSED) | A 2 MB Markdown document is searchable within **120 s** of upload | CP-27 test |
| Availability (PROPOSED, staging only) | **99.5%** of minutes per month `/readyz` returns 200 | Deferred until staging host chosen (Q7) |

Measurement environment (all local SLOs): Colima VM with **4 vCPU** and **8 GB RAM**, API with
**2** Uvicorn workers, Postgres and Redis from `make up`, data from `make seed SEED=42`, real auth seam
and row-level security switched on. Results are written to `reports/load/` as target vs actual.

---

## 5. Ingest targets

**Decision:** The system accepts **2,000 readings per second** continuously. Readings wait in a queue
capped at **50,000** (about 25 seconds of buffer). If the queue is full, senders are told to wait and
retry (HTTP 429 with a wait time) rather than the system slowing down or losing data. A reading that
keeps failing is set aside in a dead-letter queue for inspection (ADR 0001).

| Target | Value |
|---|---|
| Sustained throughput | **2,000 points/second** for **15 minutes** with zero 5xx responses |
| Queue cap | **50,000 points** in `cp:telemetry`; never exceeded (checked atomically) |
| Behaviour at cap | **429** + `Retry-After` of **1 to 30 s**; whole batch rejected, nothing partially enqueued |
| Redis down | **503** + `Retry-After: 5` |
| Ingest endpoint latency | p95 **<= 400 ms** per batch of **500 points** |
| Freshness | p95 **<= 10 s**, p99 **<= 30 s** from 202 response to row queryable in `telemetry_points` |
| Rollup freshness | hourly rollups updated within **10 min** of the hour's data arriving |
| No loss | accepted points = stored points + dead-lettered points, difference **0** (Redis kept running) |
| Duplicates | re-sending the same batch **3 times** stores each point exactly **once** |
| Worker concurrency | **4** consumers per worker process by default, each reading at most **500** entries per call |
| Retries | up to **5** deliveries, backoff 0.5 s doubling to max **30 s** with jitter, then dead-letter |
| Dead-letter cap | **100,000** entries per DLQ stream |
| Overload recovery | after a 2-minute burst at **4,000 points/second**, queue returns below **5,000** within **60 s** of the burst ending |
| Raw retention (PROPOSED, Q6) | raw points **14 days**; hourly rollups **400 days**; daily rollups **5 years** |

---

## 6. AI evaluation targets (planner assistant)

**Decision:** The assistant must find the right policy passages, stick to the facts it was given, get
answers right, refuse off-topic requests, ignore planted instructions, start answering within
**1.5 seconds**, finish within **6 seconds**, and cost no more than **2 cents** per question. A
`make eval` command checks all of this against a fixed set of test questions and prints pass/fail.

### Rules (from CLAUDE.md, mandatory)
- Golden dataset lives in `evals/golden/`.
- `make eval` reports **quality**, **p95 latency** and **cost per request in dollars**, writes a
  target-vs-actual table to `reports/eval/`, and exits non-zero if any gated target fails.
- Agent loop: **max 8 tool calls**, **hard timeout 15 s**, `max_tokens` **always set** (1,024 for the
  final answer, 512 for intermediate steps, 512 for judge calls).
- Model: Anthropic Claude (exact model is open question Q2). Embeddings: hosted model (vendor is Q3).

### Targets

| Metric | Plain-English meaning | Target | Gated |
|---|---|---|---|
| Retrieval recall@5 | Share of the expected policy passages that appear in the top 5 search results, averaged over questions | **>= 0.85** | yes |
| Faithfulness | Share of statements in answers that are supported by the retrieved text or tool results (judge-scored) | **>= 0.90** | yes |
| Answer correctness | Share of in-scope questions answered correctly vs the reference; numbers must be within **2%** | **>= 0.80** | yes |
| Out-of-scope refusal | Share of off-topic questions politely refused | **>= 0.95** | yes |
| False refusal (PROPOSED) | Share of in-scope questions wrongly refused | **<= 0.05** | yes |
| Prompt-injection resistance | Share of injection cases where the planted instruction is ignored | **= 1.00** (security rule) | yes |
| Citation presence (PROPOSED) | Share of in-scope answers with at least one citation | **>= 0.95** | yes |
| Time to first answer token | From API receiving the question to the first streamed answer token, p95 | **<= 1.5 s** | yes |
| Total time | From question received to `done` event, p95 | **<= 6 s** | yes |
| Cost per question | Model + embedding tokens x dated price table, p95 | **<= $0.02** | yes |
| Tool-call cap | Share of runs with 8 or fewer tool calls | **= 1.00** | yes |
| Hard timeout | Share of runs ending within 15 s | **= 1.00** | yes |
| Judge reliability (PROPOSED) | Agreement of the AI judge with 30 human-labelled answers | **>= 0.85** | report only |

### Golden dataset (in `evals/golden/`)

| File | Contents | Size |
|---|---|---|
| `retrieval.jsonl` | question + ids of the passages that should be found | **100** questions |
| `planner_qa.jsonl` | in-scope questions with reference answers: 40 policy-only, 30 live-data, 30 mixed | **100** questions |
| `out_of_scope.jsonl` | off-topic requests (recipes, legal advice, other tenants' data, code writing) | **40** questions |
| `injection.jsonl` | planted instructions: 10 inside policy documents, 10 inside user questions or data fields | **20** cases |
| `human_labels.jsonl` | 30 answers labelled by a person to calibrate the judge | **30** items |

Live-data answers are reproducible because `make eval` seeds a separate `cp_eval` database with
`SEED=42`. Estimated cost of one full `make eval` run: about **260 questions x $0.02 = $5.20** plus
judge calls, under **$10 per run**. Judge cost is reported separately and not counted in the
per-question cost.

### Forecast backtest targets (PROPOSED, needs approval, Q4)

Plain English: **WAPE** (weighted absolute percentage error) = total size of the misses divided by
total actual demand. 8% WAPE on a site drawing 1,000 kW means the forecast is off by about 80 kW on
average. **Coverage** = how often the real value lands inside the stated range.

| Metric | 1-30 days ahead | 31-90 days ahead |
|---|---|---|
| Peak kW WAPE | **<= 8%** | **<= 12%** |
| RU used WAPE | **<= 5%** | **<= 8%** |
| 80% interval coverage | between **75% and 85%** | between **75% and 85%** |
| 95% interval coverage | between **90% and 98%** | between **90% and 98%** |
| Improvement over seasonal naive baseline (WAPE) | at least **15%** lower | at least **15%** lower |
| MAPE | reported, not gated (unstable near zero) | reported, not gated |

Backtest method: rolling origins every **14 days** over the latest **180 days** (at least **6**
origins), each forecasting **90 days** (ADR 0006). A model that does not beat the baseline does not ship.

---

## 7. Engineering standards and the named test that proves each one

**Decision:** Every rule in CLAUDE.md has at least one automated test with a fixed name. If someone
breaks the rule, that test fails. QA writes a target-vs-actual table of these in `reports/`.

| Standard | Named test | Proves | Ticket |
|---|---|---|---|
| Backpressure | `test_enqueue_is_atomic_and_never_exceeds_cap` | 100 parallel producers never push the stream past 50,000 | CP-08 |
| Backpressure | `test_ingest_returns_429_with_retry_after_when_queue_full` | Full queue -> 429, `Retry-After` in 1..30 s, nothing enqueued | CP-19 |
| Backpressure | `test_worker_concurrency_never_exceeds_configured_limit` | At most N messages processed at once | CP-16 |
| Backpressure | `test_planner_returns_429_when_concurrency_limit_reached` | Assistant concurrency bounded per API instance and per tenant | CP-47 |
| Retries | `test_retry_uses_exponential_backoff_with_jitter` | Delays grow 0.5, 1, 2, 4 s (+/- jitter), capped at 30 s | CP-09 |
| Retries | `test_retry_stops_after_max_attempts` | Gives up after the configured maximum | CP-09 |
| Retries | `test_every_outbound_call_has_timeout` | No HTTP/LLM/embedding client is created without a timeout (code scan + runtime) | CP-09 |
| Retries | `test_idempotency_key_replay_returns_original_response` | Repeat POST returns the same response, no second row | CP-17 |
| Retries | `test_idempotency_key_reuse_with_different_body_returns_422` | Key misuse detected | CP-17 |
| Partial failure | `test_poison_message_moves_to_dead_letter_after_5_deliveries` | Bad item goes to DLQ with error and trace id; others continue | CP-08, CP-20 |
| Partial failure | `test_outbox_event_published_only_after_commit` | Rolled-back change publishes nothing; committed change publishes once | CP-16 |
| Partial failure | `test_outbox_relay_crash_does_not_lose_or_duplicate_effects` | Kill relay mid-batch; every event's effect happens exactly once | CP-16 |
| Partial failure | `test_batch_ingest_returns_per_item_status` | Mixed good/bad batch returns status per item | CP-19 |
| Pagination | `test_page_size_capped_at_100` | `limit=101` -> 422 | CP-07 |
| Pagination | `test_keyset_pagination_no_duplicates_or_gaps_under_concurrent_inserts` | Paging while inserting sees each original row exactly once | CP-07 |
| Pagination | `test_ties_broken_by_id` | Equal sort values come back in stable id order | CP-07 |
| Pagination | `test_tampered_cursor_rejected` | Edited cursor -> 400 | CP-07 |
| Pagination | `test_every_list_route_is_paginated` | Every list route has `limit`/`cursor` and enforces max 100 | CP-34 |
| Migrations | `test_all_migrations_upgrade_downgrade_roundtrip` | upgrade -> downgrade -> upgrade on seeded data | CP-05 (extended by CP-13) |
| Migrations | `test_single_alembic_head` | Exactly one migration head | CP-05 |
| Migrations | `test_no_destructive_ops_outside_contract_migrations` | No drops/renames in expand migrations | CP-05 |
| Migrations | `test_backfill_is_batched_and_resumable` | Backfill stops at batch k, restarts, finishes with no duplicates | CP-42 |
| Tenancy | `test_every_tenant_table_has_tenant_id_and_forced_rls` | Schema scan of every table | CP-05 |
| Tenancy | `test_rls_blocks_query_without_tenant_setting` | Query without `app.tenant_id` errors | CP-05 |
| Tenancy | `test_app_role_cannot_bypass_rls` | `cp_app` cannot see other tenants, even reading partitions | CP-05, CP-13 |
| Tenancy | `test_child_cannot_reference_other_tenant_parent` | Composite foreign keys hold | CP-05 |
| Tenancy | `test_cross_tenant_access_returns_404_for_every_endpoint` | Each endpoint tried with another tenant's id | CP-34 (plus per-endpoint tests in each ticket) |
| Tenancy | `test_agent_tools_cannot_read_other_tenant` | Tools under tenant A never return tenant B data | CP-36 |
| Observability | `test_logs_are_json_with_trace_id` | Every log line is JSON with `trace_id` | CP-06 |
| Observability | `test_trace_spans_api_to_queue_to_worker` | One trace id spans API, stream entry and worker span | CP-20 |
| Observability | `test_trace_continues_into_llm_call` | Planner trace includes LLM spans with token and cost attributes | CP-43 |
| Security | `test_every_route_requires_auth_unless_public` | Only `/healthz` and `/readyz` are public | CP-12 |
| Security | `test_every_route_has_authorization_test` | Every route appears in an authorization test, including a cross-tenant case | CP-12 |
| Security | `test_dev_auth_refuses_non_local_env` | Fake login cannot run outside local/test | CP-12 |
| Security | `test_security_headers_present` and `test_rate_limit_per_tenant` | Headers, CORS allow-list, 429 on abuse | CP-26 |
| Security | `check-images.sh: containers_run_as_non_root` | Every image runs as a non-root user, read-only root filesystem | CP-15 |
| Security | `check-actions-pinned.sh: actions_pinned_to_sha` | Every GitHub Action pinned to a 40-character commit SHA | CP-41 |
| AI rules | `test_agent_stops_after_8_tool_calls` | 9th tool call is never executed | CP-43 |
| AI rules | `test_agent_hard_timeout_15s` | Slow tool -> loop ends by 15 s with `timed_out` | CP-43 |
| AI rules | `test_every_llm_call_sets_max_tokens` | Code scan + runtime check on the client wrapper | CP-43 |
| AI rules | `test_cost_guard_stops_before_limit` | Loop stops before projected cost exceeds $0.02 | CP-43 |
| AI rules | `test_untrusted_content_is_wrapped_and_tool_args_validated` | Injection defenses in place | CP-32 |
| AI rules | `test_make_eval_reports_quality_latency_cost` | `make eval` output contains all gated metrics in dollars and seconds | CP-37 |

---

## 8. Risks

**Decision:** The biggest risks are (1) the AI cost and speed targets depending on which Claude model
you choose, (2) building before the login provider is chosen, and (3) laptop limits for load tests.
Each has a mitigation below.

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Auth provider undecided; later swap causes rework | High | Medium | Swappable auth seam (ADR 0008); every endpoint already tested for 401/403/404; swap is one ticket (CP-44) |
| R2 | Larger Claude models may exceed **$0.02/question** with up to 8 tool calls | Medium | High | Cost guard stops early; prompt caching; parallel tool calls in one turn; recommend a fast, low-cost Claude model (Q2) |
| R3 | Time to first answer token **<= 1.5 s** is hard when tools must run before answering | Medium | Medium | Parallel tool calls, small tool results (<= 4,000 characters), streaming, fast model; measured in every agent PR |
| R4 | Redis crash loses up to about **1 s** of accepted telemetry (`appendfsync everysec`) | Low | Low | Accepted for sampled telemetry; business events use the Postgres outbox (ADR 0001) |
| R5 | Raw telemetry growth: at 2,000 points/second, about **173 million rows/day** | Medium | High | Daily partitions, retention job (14 days PROPOSED), rollups for long-term views; the 2,000/s rate is a burst/headroom target, not the synthetic steady state |
| R6 | pgvector search filtered by tenant can return fewer than 5 results | Medium | Medium | pgvector iterative index scans, hybrid keyword search, recall@5 gate in `make eval` |
| R7 | Synthetic data is cleaner than reality, so forecast accuracy looks better than it would be | High | Medium | Generator adds gaps, bad rows and step changes; targets are PROPOSED and revisited with real data |
| R8 | Prompt injection through uploaded policy documents | Medium | High | Untrusted-content wrapping, read-only allow-listed tools, tenant never a tool argument, 20 injection cases at 100% target |
| R9 | Two approvals over-allocate the same room | Medium | High | Row lock on room during check; concurrency test with 20 parallel approvals |
| R10 | Laptop load tests are noisy, giving flaky SLO results | Medium | Low | Fixed Colima size, warm-up, 3 runs with the median reported |
| R11 | Embedding vendor not chosen, blocking real document search (CP-27) | Medium | Medium | Interface + deterministic fake for tests; real adapter needs Q3 answered before Wave 4 |
| R12 | Dependencies added early show up in vulnerability scans | Medium | Low | Trivy gate already in CI; Dependabot (CP-41); pin versions |
| R13 | About 50 PRs need director approval | High | Medium | Small PRs (under 800 lines), plain-English summaries, waves let several merge per sitting |
| R14 | Forecast and capacity math defined inconsistently across API, web and assistant | Medium | Medium | Single definitions in docs/metrics.md and one capacity service used by all |

---

## 9. Tickets by wave

**Decision:** The work is split into **52 tickets** in **9 waves**. Tickets in the same wave can be built
at the same time because **no two of them touch the same file** (the file list for each ticket proves
it). A ticket only depends on tickets from earlier waves. Two tickets are blocked on your decisions
(CP-44 auth, CP-49 staging).

Rules for every ticket:
- Size: estimated changed lines exclude lock files and generated files (ADR 0005). If a ticket grows
  past about 800 lines, the owner stops and proposes a split.
- Risk tier from CLAUDE.md: high = auth, tenancy, migrations, queues, AI agent loop, CI/CD, infra,
  dependencies; medium = features and endpoints; low = docs, UI text, tests only. Security agent PRs are
  always risk:high.
- Every endpoint ticket includes: its OpenAPI path and component files, an authorization test per route
  (with a cross-tenant attempt), and a list entry declaring the routes it covers (`AUTHZ_COVERED`).
- Every ticket ends with `make check` and `make test` passing.
- Repository layout and parallel-safety rules: ADR 0005.

### Wave 1: foundations

Overlap check: CP-01 = `infra/compose`, `infra/postgres`, `infra/redis`, `infra/scripts/check-stack.sh`, `.env.example`;
CP-02 = root `pyproject.toml`, `uv.lock`, `Makefile`, package skeletons; CP-03 = `docs/threat-model.md`;
CP-04 = `docs/metrics.md`. No shared files.

#### CP-01: Local data stack (Postgres + pgvector, Redis), database roles, environment template
- Owner: release | Risk: risk:high (infra) | Size: ~250 lines | Depends on: none
- Files: `infra/compose/compose.yaml`, `infra/postgres/init/01-roles.sql`, `infra/postgres/init/02-extensions.sql`, `infra/redis/redis.conf`, `infra/scripts/check-stack.sh`, `.env.example`
- Acceptance:
  1. `docker compose -f infra/compose/compose.yaml up -d` starts Postgres 16 with pgvector on host port **15432** and Redis 7 on host port **16379**; both healthchecks pass within **60 s**.
  2. Roles `cp_owner`, `cp_app` (NOBYPASSRLS, no CREATE) and `cp_scheduler` exist; `vector` extension is installed; database `cp_eval` also exists.
  3. Redis runs with `appendonly yes`, `appendfsync everysec`, `maxmemory-policy noeviction` (verified by `CONFIG GET` in `check-stack.sh`).
  4. `.env.example` lists every planned variable name with empty or dummy values only (DB URLs per role, `REDIS_URL`, `CP_ENV`, `CP_AUTH_PROVIDER`, `CP_AUTH_DEV_SIGNING_KEY`, `CURSOR_SIGNING_KEY`, `ANTHROPIC_API_KEY`, `CP_AGENT_MODEL`, `CP_EMBED_PROVIDER`, `CP_EMBED_MODEL`, `CP_EMBED_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT`, stream caps, worker concurrency).
  5. Both images are multi-arch (linux/arm64 and linux/amd64), pinned by digest; containers run as non-root users.
  6. No host ports other than 15432, 16379, 18000 and 18080 are used by the stack.

#### CP-02: Python workspace, dependency baseline, Makefile, lint and type-check config
- Owner: builder | Risk: risk:high (dependencies) | Size: ~550 lines | Depends on: none
- Files: `pyproject.toml`, `uv.lock`, `.python-version`, `Makefile`, and for each of `core`, `api`, `worker`, `forecast`, `agent`, `seed`, `evals`, `load`, `tests`: `<pkg>/pyproject.toml`, `<pkg>/src/cp_<pkg>/__init__.py` (where applicable), `<pkg>/tests/conftest.py`, `<pkg>/tests/test_smoke.py`; `worker/src/cp_worker/__main__.py` (idle placeholder); `core/src/cp_core/settings.py`; `core/src/cp_core/testing/__init__.py`, `core/src/cp_core/testing/containers.py` (Testcontainers Postgres and Redis fixtures)
- Acceptance:
  1. `uv sync` installs the workspace on arm64 and amd64 with Python 3.12; `uv.lock` committed.
  2. Baseline dependencies declared with a one-line reason each in the PR: FastAPI, Uvicorn, Pydantic v2, pydantic-settings, SQLAlchemy 2, Alembic, psycopg 3, redis-py, httpx, OpenTelemetry (API, SDK, FastAPI, SQLAlchemy, httpx, Redis instrumentations), statsmodels, numpy, pandas, anthropic, pgvector (Python), python-multipart, PyJWT; dev: pytest, pytest-asyncio, Testcontainers, ruff, mypy, Locust, hypothesis.
  3. `make check` runs ruff and mypy (strict on `core`, `api`, `agent`) and, when `package.json` exists, `pnpm turbo lint typecheck`; it passes on the skeleton.
  4. `make test` runs pytest across all workspace members and, when present, `pnpm turbo test`; it passes.
  5. Makefile has targets `up`, `down`, `check`, `test`, `eval`, `migrate`, `seed`, `dev-token`, `load`, `backtest`, `check-images`; `make up` runs compose from `infra/compose/compose.yaml` then `make migrate`; `make eval` prints "no evals yet" and exits 0 until `cp_evals` has a runner.
  6. Shared fixtures start throwaway Postgres (pgvector) and Redis containers for tests in under **30 s**.

#### CP-03: Threat model v1
- Owner: security | Risk: risk:high (security agent rule) | Size: ~400 lines | Depends on: none
- Files: `docs/threat-model.md`
- Acceptance:
  1. Lists assets, actors (5 roles plus anonymous and compromised-sender), entry points (every endpoint in spec section 3, queues, worker, LLM and embedding calls, uploads), and trust boundaries.
  2. For each entry point, STRIDE threats (spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege) with mitigation and the named test that proves it.
  3. Marks `/healthz` and `/readyz` as the only public endpoints.
  4. Has a section on AI threats: prompt injection via documents and questions, tool misuse, data exfiltration, cost abuse.
  5. Records "auth provider undecided" with the ADR 0008 compensating controls.

#### CP-04: Metric definitions
- Owner: analytics | Risk: risk:low (docs) | Size: ~250 lines | Depends on: none
- Files: `docs/metrics.md`
- Acceptance:
  1. Defines capacity, committed, used, headroom (kW and RU, room and site level), utilization %, peak kW, WAPE, MAPE, interval coverage, recall@5, faithfulness, correctness, cost per question; each with name, plain-English meaning, formula, grain, owner.
  2. States that forecasts model organic demand and committed reservations are layered on top (ADR 0006).
  3. Each metric names the code module that will compute it (to be filled by later tickets).

### Wave 2: platform core

Overlap check: CP-05 = `core/src/cp_core/db/**`, `core/migrations/**`, `core/alembic.ini`, `core/src/cp_core/testing/db.py`, `core/tests/db/**`;
CP-06 = `api/src/cp_api/{main,app_factory,discovery,problems}.py`, `api/src/cp_api/{routers,plugins,schemas,contract}/` initial files, `core/src/cp_core/observability/**`, `contracts/openapi/{base.yaml,paths/health.yaml,components/problems.yaml}`, `api/tests/core/**`, `core/tests/observability/**`;
CP-07 = `core/src/cp_core/pagination.py`, `core/tests/test_pagination.py`, `contracts/openapi/components/pagination.yaml`;
CP-08 = `core/src/cp_core/streams/**`, `core/tests/streams/**`;
CP-09 = `core/src/cp_core/resilience/**`, `core/tests/resilience/**`;
CP-10 = `forecast/src/cp_forecast/{series,baseline,model,backtest,metrics}.py`, `forecast/tests/model/**`, `reports/backtest/**`;
CP-11 = `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `turbo.json`, `apps/web/**`. No shared files.

#### CP-05: Core schema, tenant isolation and migration 0001
- Owner: data-engineer | Risk: risk:high (migrations, tenancy) | Size: ~750 lines | Depends on: CP-01, CP-02
- Files: `core/alembic.ini`, `core/migrations/env.py`, `core/migrations/script.py.mako`, `core/migrations/versions/0001_core_schema.py`, `core/src/cp_core/db/__init__.py`, `core/src/cp_core/db/engine.py`, `core/src/cp_core/db/session.py`, `core/src/cp_core/db/base.py`, `core/src/cp_core/db/models/__init__.py` (auto-imports model modules), `core/src/cp_core/db/models/{tenancy,inventory,reservations,platform}.py`, `core/src/cp_core/testing/db.py`, `core/tests/db/**`
- Acceptance:
  1. Migration 0001 creates `tenants`, `sites`, `rooms`, `racks`, `reservations`, `allocations`, `reservation_events`, `outbox_events`, `processed_events`, `idempotency_keys`, `audit_log`, `job_checkpoints` exactly as in spec section 2, with composite tenant foreign keys and indexes leading on `tenant_id`.
  2. RLS is enabled and forced on every tenant-owned table with `USING` and `WITH CHECK` on `app.tenant_id`.
  3. `tenant_session(tenant_id)` sets `SET LOCAL app.tenant_id` per transaction; using a session without it raises.
  4. Named tests pass: `test_every_tenant_table_has_tenant_id_and_forced_rls`, `test_rls_blocks_query_without_tenant_setting`, `test_app_role_cannot_bypass_rls`, `test_child_cannot_reference_other_tenant_parent`, `test_all_migrations_upgrade_downgrade_roundtrip`, `test_single_alembic_head`, `test_no_destructive_ops_outside_contract_migrations`.
  5. Migration sets `lock_timeout = 5s` and `statement_timeout = 60s`; PR shows upgrade and downgrade output and `EXPLAIN` for the list-query indexes.

#### CP-06: API skeleton, auto-discovery, error format, JSON logs and tracing, OpenAPI scaffolding
- Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-02
- Files: `api/src/cp_api/main.py`, `api/src/cp_api/app_factory.py`, `api/src/cp_api/discovery.py`, `api/src/cp_api/problems.py`, `api/src/cp_api/routers/__init__.py`, `api/src/cp_api/routers/health.py`, `api/src/cp_api/plugins/__init__.py`, `api/src/cp_api/schemas/__init__.py`, `api/src/cp_api/contract/__init__.py`, `api/src/cp_api/contract/bundle.py`, `core/src/cp_core/observability/{__init__,logging,tracing}.py`, `contracts/openapi/base.yaml`, `contracts/openapi/paths/health.yaml`, `contracts/openapi/components/problems.yaml`, `api/tests/core/**`, `core/tests/observability/**`
- Acceptance:
  1. Every module in `cp_api/routers/` and `cp_api/plugins/` is loaded automatically; `test_route_inventory_matches_contract` fails if a route is missing from the bundled contract or vice versa.
  2. `bundle.py` merges `base.yaml` + `paths/*.yaml` + `components/*.yaml` into `contracts/openapi/dist/openapi.json` (git-ignored) and validates it as OpenAPI 3.1.
  3. All errors return RFC 9457 problem+json with `trace_id`; unhandled exceptions return 500 with no stack trace (tested).
  4. `test_logs_are_json_with_trace_id` passes; incoming `traceparent` is continued and `X-Trace-Id` returned.
  5. `GET /healthz` returns 200 in under **50 ms**; `GET /readyz` returns 503 when Postgres or Redis is unreachable.

#### CP-07: Keyset pagination helper (ADR 0002)
- Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-02
- Files: `core/src/cp_core/pagination.py`, `core/tests/test_pagination.py`, `contracts/openapi/components/pagination.yaml`
- Acceptance:
  1. Helper builds `WHERE (sort_key, id) > (...) ORDER BY sort_key, id LIMIT limit+1` for ascending and descending sorts and returns `{items, next_cursor}`.
  2. Cursors are base64url JSON with HMAC-SHA256 over sort, direction, last values and filter hash.
  3. Named tests pass: `test_page_size_capped_at_100`, `test_keyset_pagination_no_duplicates_or_gaps_under_concurrent_inserts` (10,000 rows, inserts during paging), `test_ties_broken_by_id`, `test_tampered_cursor_rejected`.
  4. Default limit is **50**; limit outside 1..100 raises a validation error mapped to 422.

#### CP-08: Redis Streams library: bounded producer, consumer groups, dead-letter, reclaim
- Owner: builder | Risk: risk:high (queues) | Size: ~650 lines | Depends on: CP-01, CP-02
- Files: `core/src/cp_core/streams/{__init__,settings,producer,consumer,dlq,cli}.py`, `core/src/cp_core/streams/lua/enqueue_bounded.lua`, `core/tests/streams/**`
- Acceptance:
  1. `enqueue_batch(stream, entries)` appends all entries or none, atomically refusing if `XLEN + n > cap`; returns current depth and a suggested `Retry-After` of **1..30 s**. `test_enqueue_is_atomic_and_never_exceeds_cap` passes with 100 concurrent producers.
  2. Consumer base reads with a consumer group, at most **500** entries per read, acknowledges and deletes on success, and reclaims entries idle for more than **60 s**.
  3. Failures retry with backoff (0.5 s doubling, max 30 s, jitter); after **5** deliveries the entry moves to `cp:dlq:<stream>` with error, attempts and `traceparent`; `test_poison_message_moves_to_dead_letter_after_5_deliveries` passes.
  4. DLQ capped at **100,000** entries; `python -m cp_core.streams.cli replay <stream> [--limit N]` re-queues DLQ entries respecting the cap.
  5. Each entry carries `tenant_id` and `traceparent`; consumer spans link to the producer span.

#### CP-09: Resilience library: retries, timeouts, outbound HTTP client
- Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-02
- Files: `core/src/cp_core/resilience/{__init__,retry,http,settings}.py`, `core/tests/resilience/**`
- Acceptance:
  1. `retry(...)` implements exponential backoff with full jitter, max attempts, max delay, and retries only on retryable errors (timeouts, 429, 5xx); honours `Retry-After`.
  2. `make_http_client(...)` requires connect and total timeouts (defaults **2 s** connect, **10 s** total) and adds an `Idempotency-Key` to retried non-GET requests.
  3. Named tests pass: `test_retry_uses_exponential_backoff_with_jitter`, `test_retry_stops_after_max_attempts`, `test_every_outbound_call_has_timeout` (fails if `httpx.Client(`, `httpx.AsyncClient(` or `anthropic.Anthropic(` appears outside this module).

#### CP-10: Forecast model, seasonal naive baseline and backtest harness (ADR 0006)
- Owner: analytics | Risk: risk:medium | Size: ~700 lines | Depends on: CP-02
- Files: `forecast/src/cp_forecast/{series,baseline,model,backtest,metrics}.py`, `forecast/tests/model/**`, `reports/backtest/README.md`, `reports/backtest/<date>-synthetic.md`
- Acceptance:
  1. `fit_forecast(series, horizon=90)` returns 90 daily points with `yhat`, `lo80`, `hi80`, `lo95`, `hi95`; uses ETS damped + weekly seasonality, MSTL + ETS at >= **730** days, refuses under **56** days.
  2. Seasonal naive baseline and rolling-origin backtest (every 14 days over 180 days, >= 6 origins) report WAPE, MAPE and coverage by horizon bucket.
  3. On the synthetic test series (fixed seed) the model beats the baseline WAPE by >= **15%** and meets the PROPOSED targets in spec section 6, or the PR states the gap in plain English.
  4. One site-metric fit completes in under **1 s** on the reference laptop.
  5. `make backtest` writes a target-vs-actual table to `reports/backtest/`.

#### CP-11: Web shell (pnpm + Turborepo + Next.js)
- Owner: builder | Risk: risk:high (dependencies) | Size: ~600 lines | Depends on: CP-02
- Files: `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `turbo.json`, `apps/web/**` (layout, navigation with Sites, Reservations, Forecasts, Documents, Planner placeholder pages, API base config, Vitest config, Playwright config, `apps/web/e2e/smoke.spec.ts`)
- Acceptance:
  1. `pnpm install` and `pnpm turbo build lint typecheck test` pass; lock file committed.
  2. Dependencies front-loaded with reasons: Next.js, React, TypeScript, Vitest, Playwright, openapi-typescript, openapi-fetch, a chart library (Recharts), Zod.
  3. Dev server runs on port **18080**; navigation shows all 5 sections with placeholder pages.
  4. `apps/web/package.json` has a `gen:api` script that generates types from `contracts/openapi/dist/openapi.json`.
  5. Playwright smoke test loads every section without console errors.

### Wave 3: auth seam, data model extension, runtime services

Overlap check: CP-12 = `api/src/cp_api/auth/**`, `api/src/cp_api/plugins/auth.py`, `contracts/openapi/components/security.yaml`, `api/tests/auth/**`, `api/tests/security/test_route_auth_inventory.py`, `docs/threat-model.md`;
CP-13 = `core/src/cp_core/db/models/{telemetry,forecasting,knowledge,planner}.py`, `core/src/cp_core/telemetry/{__init__,messages}.py`, `core/migrations/versions/0002_*.py`, `core/tests/db/test_0002_schema.py`;
CP-14 = `seed/src/cp_seed/{generate,inventory,series,load_inventory,cli}.py`, `seed/tests/inventory/**`;
CP-15 = `infra/docker/**`, `.dockerignore`, `infra/compose/compose.yaml`, `infra/scripts/check-images.sh`;
CP-16 = `worker/src/cp_worker/{__main__,runner,discovery,outbox_relay,inbox}.py`, `worker/src/cp_worker/{consumers,schedules,jobs}/__init__.py`, `worker/src/cp_worker/schedules/scheduler.py`, `core/src/cp_core/outbox.py`, `worker/tests/runtime/**`;
CP-17 = `api/src/cp_api/idempotency.py`, `contracts/openapi/components/idempotency.yaml`, `api/tests/idempotency/**`;
CP-18 = `core/src/cp_core/services/{__init__,capacity}.py`, `core/tests/services/test_capacity.py`, `docs/metrics.md`. No shared files.

#### CP-12: Swappable auth seam, deny-by-default, permission matrix, authorization test harness (ADR 0008)
- Owner: security | Risk: risk:high (auth) | Size: ~650 lines | Depends on: CP-06
- Files: `api/src/cp_api/auth/{__init__,context,provider,dev_provider,dev_token,permissions}.py`, `api/src/cp_api/plugins/auth.py`, `contracts/openapi/components/security.yaml`, `api/tests/auth/**`, `api/tests/security/test_route_auth_inventory.py`, `docs/threat-model.md`
- Acceptance:
  1. Every request passes through `AuthProvider.authenticate` and yields `AuthContext(tenant_id, user_id, roles)`; missing or invalid token -> 401; `require_role(...)` -> 403.
  2. `dev` provider validates HS256 tokens with **15-minute** expiry; `make dev-token TENANT=<slug> ROLES=<list>` prints one; `test_dev_auth_refuses_non_local_env` passes.
  3. `test_every_route_requires_auth_unless_public` passes (only `/healthz`, `/readyz` public).
  4. `test_every_route_has_authorization_test` collects `AUTHZ_COVERED` declarations from all test modules and fails for any uncovered route.
  5. Permission matrix in spec section 2 implemented as data in `permissions.py`, with a test per row.
  6. Threat model updated; tokens are never logged (test asserts redaction).

#### CP-13: Migration 0002: telemetry, rollups, forecasts, documents, planner history
- Owner: data-engineer | Risk: risk:high (migrations, tenancy) | Size: ~700 lines | Depends on: CP-05
- Files: `core/src/cp_core/db/models/{telemetry,forecasting,knowledge,planner}.py`, `core/src/cp_core/telemetry/__init__.py`, `core/src/cp_core/telemetry/messages.py`, `core/migrations/versions/0002_telemetry_forecast_knowledge.py`, `core/tests/db/test_0002_schema.py`
- Acceptance:
  1. Creates `telemetry_points` (partitioned by day), `telemetry_hourly`, `telemetry_daily`, `forecast_runs`, `forecasts`, `forecast_points`, `policy_documents`, `document_chunks` (`vector(1024)`, HNSW cosine, GIN on tsv), `planner_questions`, all tenant-owned with forced RLS.
  2. Creates partitions from 7 days back to 14 days ahead and a SQL function `cp_create_telemetry_partitions(days_ahead int)`; `cp_app` has no privileges on individual partitions.
  3. `messages.py` defines the versioned telemetry stream message (`v`, `tenant_id`, `rack_id`, `room_id`, `site_id`, `metric`, `ts`, `value`, `batch_id`, `traceparent`) with Pydantic validation.
  4. Existing tenancy and migration named tests now cover the new tables and pass; upgrade/downgrade round trip passes on seeded data.
  5. PR shows `EXPLAIN` proving index use for `(tenant_id, rack_id, metric, ts)` and `(tenant_id, room_id, metric, day)` lookups.

#### CP-14: Synthetic data generator and inventory/reservation loader
- Owner: data-engineer | Risk: risk:medium | Size: ~650 lines | Depends on: CP-05
- Files: `seed/src/cp_seed/{generate,inventory,series,load_inventory,cli}.py`, `seed/tests/inventory/**`
- Acceptance:
  1. `make seed SEED=42` loads 3 tenants, 12 sites, 4-8 rooms per site, 20-60 racks per room, and about 150 reservations across all states; identical results on every run with the same seed.
  2. `series.py` produces deterministic daily and per-minute kW and RU series with trend, weekly and yearly seasonality, step changes, 1% gaps and 0.5% bad rows (pure functions, no DB).
  3. Loader is idempotent: running it twice produces no duplicates (row counts equal).
  4. Data quality tests: row counts per table, no nulls in required fields, referential integrity, every reservation's committed kW within room capacity.
  5. The CLI auto-discovers `load_*.py` modules so later loaders need no CLI edits.

#### CP-15: Container images and app services in compose
- Owner: release | Risk: risk:high (infra) | Size: ~400 lines | Depends on: CP-01, CP-06, CP-11
- Files: `infra/docker/api.Dockerfile`, `infra/docker/worker.Dockerfile`, `infra/docker/web.Dockerfile`, `.dockerignore`, `infra/compose/compose.yaml`, `infra/scripts/check-images.sh`
- Acceptance:
  1. Images build for linux/arm64 and linux/amd64 with `docker buildx`; base images pinned by digest; slim or distroless runtime.
  2. Containers run as a non-root user with read-only root filesystem and all Linux capabilities dropped; `make check-images` verifies this and fails otherwise.
  3. `make up` starts api on host port **18000**, web on **18080**, worker with no host port; all healthy within **90 s** on the reference laptop.
  4. No secrets in image layers (checked by Trivy in CI); Trivy finds zero unfixed HIGH or CRITICAL issues.

#### CP-16: Worker runtime, transactional outbox and outbox relay
- Owner: builder | Risk: risk:high (queues) | Size: ~650 lines | Depends on: CP-05, CP-08
- Files: `worker/src/cp_worker/{__main__,runner,discovery,outbox_relay,inbox}.py`, `worker/src/cp_worker/consumers/__init__.py`, `worker/src/cp_worker/schedules/__init__.py`, `worker/src/cp_worker/schedules/scheduler.py`, `worker/src/cp_worker/jobs/__init__.py`, `core/src/cp_core/outbox.py`, `worker/tests/runtime/**`
- Acceptance:
  1. Worker auto-discovers consumers in `consumers/` and schedules in `schedules/`; concurrency per consumer is configurable (default **4**); `test_worker_concurrency_never_exceeds_configured_limit` passes.
  2. `outbox.add(session, stream, event_type, payload)` writes to `outbox_events` in the caller's transaction; relay publishes in batches of **100** with `FOR UPDATE SKIP LOCKED` and marks published.
  3. `test_outbox_event_published_only_after_commit` and `test_outbox_relay_crash_does_not_lose_or_duplicate_effects` pass.
  4. `inbox.once(consumer, event_id)` records processed events so repeats are skipped.
  5. Scheduler uses a Redis lock so only one worker runs each scheduled job; graceful shutdown finishes in-flight items within **10 s**.

#### CP-17: Idempotency keys for API writes
- Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-05, CP-06
- Files: `api/src/cp_api/idempotency.py`, `contracts/openapi/components/idempotency.yaml`, `api/tests/idempotency/**`
- Acceptance:
  1. A reusable dependency stores `(tenant_id, key)` with request hash and response for **24 h**; `POST` without the header -> 422.
  2. `test_idempotency_key_replay_returns_original_response` and `test_idempotency_key_reuse_with_different_body_returns_422` pass; concurrent duplicate in flight -> 409.
  3. The same key used by two different tenants does not collide (tested).
  4. Expired keys are purged by a batched delete function (5,000 rows per batch).

#### CP-18: Capacity and headroom service
- Owner: analytics | Risk: risk:medium | Size: ~450 lines | Depends on: CP-04, CP-05
- Files: `core/src/cp_core/services/__init__.py`, `core/src/cp_core/services/capacity.py`, `core/tests/services/test_capacity.py`, `docs/metrics.md`
- Acceptance:
  1. Computes per-room and per-site capacity, committed (allocated + approved-unallocated) and headroom for kW and RU exactly as defined in docs/metrics.md; "used" is an optional input supplied later by the utilization service.
  2. `check_fits(room or rack, kw, ru)` returns fit/no-fit with the limiting dimension; used by reservations.
  3. Tests cover empty rooms, full rooms, archived racks, released allocations and rack-level vs room-level requests.
  4. Room-level capacity query runs in under **20 ms** on seeded data (PR shows `EXPLAIN ANALYZE`).
  5. docs/metrics.md updated with the module path for each capacity metric.

### Wave 4: product features (ingest, inventory, reservations, forecasts read, documents)

Overlap check: CP-19 = `api/src/cp_api/routers/telemetry_ingest.py`, `api/src/cp_api/schemas/telemetry.py`, `core/src/cp_core/telemetry/rack_cache.py`, `contracts/openapi/{paths/telemetry-ingest.yaml,components/telemetry.yaml}`, `api/tests/telemetry/**`;
CP-20 = `worker/src/cp_worker/consumers/telemetry.py`, `worker/src/cp_worker/schedules/rollups.py`, `core/src/cp_core/telemetry/{writer,rollups}.py`, `core/src/cp_core/services/utilization.py`, `core/tests/telemetry/**`, `worker/tests/telemetry/**`;
CP-21 = `api/src/cp_api/routers/{sites,rooms}.py`, `api/src/cp_api/schemas/{sites,rooms}.py`, `core/src/cp_core/services/{sites,rooms}.py`, `contracts/openapi/{paths,components}/{sites,rooms}.yaml`, `api/tests/{sites,rooms}/**`;
CP-22 = `api/src/cp_api/routers/racks.py`, `api/src/cp_api/schemas/racks.py`, `core/src/cp_core/services/racks.py`, `contracts/openapi/{paths,components}/racks.yaml`, `api/tests/racks/**`;
CP-23 = `api/src/cp_api/routers/reservations.py`, `api/src/cp_api/schemas/reservations.py`, `core/src/cp_core/services/reservations.py`, `core/src/cp_core/domain/{__init__,reservation_state}.py`, `worker/src/cp_worker/schedules/reservation_expiry.py`, `worker/src/cp_worker/consumers/events_notifier.py`, `contracts/openapi/{paths,components}/reservations.yaml`, `api/tests/reservations/**`, `core/tests/services/test_reservations.py`;
CP-24 = `core/src/cp_core/services/forecasts.py`, `api/src/cp_api/routers/forecasts.py`, `api/src/cp_api/schemas/forecasts.py`, `contracts/openapi/{paths,components}/forecasts.yaml`, `api/tests/forecasts/**`;
CP-25 = `seed/src/cp_seed/load_telemetry.py`, `seed/policies/*.md`, `seed/tests/telemetry/**`;
CP-26 = `api/src/cp_api/plugins/{rate_limit,security_headers,cors,body_limit}.py`, `api/src/cp_api/security/**`, `api/tests/security/test_{rate_limit,headers,cors,body_limit}.py`, `docs/threat-model.md`;
CP-27 = `api/src/cp_api/routers/documents.py`, `api/src/cp_api/schemas/documents.py`, `contracts/openapi/{paths,components}/documents.yaml`, `worker/src/cp_worker/consumers/documents.py`, `core/src/cp_core/ai/{__init__,embeddings,chunking}.py`, `api/tests/documents/**`, `worker/tests/documents/**`, `core/tests/ai/**`. No shared files.

#### CP-19: Telemetry ingest endpoint with backpressure and per-item status
- Owner: builder | Risk: risk:high (queues) | Size: ~550 lines | Depends on: CP-08, CP-12, CP-13, CP-17
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/telemetry/batches` (role `ingestor`) validates 1-500 points, body <= 256 KB, `ts` window (+5 min / -7 days), value ranges, and rack ownership via a per-tenant rack-id cache refreshed every **60 s**.
  2. Returns **202** with per-item status; `test_batch_ingest_returns_per_item_status` passes.
  3. Full queue -> **429** + `Retry-After` 1..30 s and nothing enqueued (`test_ingest_returns_429_with_retry_after_when_queue_full`); Redis down -> **503** + `Retry-After: 5`.
  4. `Idempotency-Key` replay within 24 h returns the original response without re-enqueueing (stored in Redis).
  5. Endpoint p95 <= **400 ms** for 500-point batches in a local micro-benchmark included in the PR.
  6. Authorization tests: non-ingestor -> 403; rack from another tenant -> item `rejected` with reason `unknown_rack`.

#### CP-20: Telemetry consumer, rollups and utilization service
- Owner: data-engineer | Risk: risk:high (queues) | Size: ~700 lines | Depends on: CP-13, CP-16
- Files: as in overlap check
- Acceptance:
  1. Consumer writes points with multi-row `INSERT ... ON CONFLICT DO NOTHING` in batches of up to **500**, under the message's tenant; re-delivering a batch 3 times stores each point once.
  2. Invalid or failing messages retry, then dead-letter after 5 deliveries; valid messages in the same read still commit.
  3. Rollup schedule upserts hourly rollups for the last 2 hours every **5 min** and daily rollups at **00:15 UTC**; idempotent on re-run.
  4. `utilization.py` returns latest and windowed used kW/RU per room and site from rollups, paginated by keyset.
  5. `test_trace_spans_api_to_queue_to_worker` passes; freshness p95 <= **10 s** at 2,000 points/second in an integration test of 60 s.

#### CP-21: Sites and rooms CRUD
- Owner: builder | Risk: risk:medium | Size: ~700 lines | Depends on: CP-07, CP-12, CP-17
- Files: as in overlap check
- Acceptance:
  1. Endpoints for sites and rooms in spec section 3 implemented with keyset pagination, idempotency on POST, and admin-only writes.
  2. Room `capacity_kw` sum may not exceed site `capacity_kw` (422); archiving a site or room with active reservations -> 409.
  3. Unique site code per tenant and room name per site enforced (409 on duplicate).
  4. Authorization tests per route including cross-tenant -> 404; `AUTHZ_COVERED` declared.
  5. Writes record `audit_log` entries.

#### CP-22: Racks CRUD
- Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-07, CP-12, CP-17
- Files: as in overlap check
- Acceptance:
  1. Rack endpoints in spec section 3 implemented with pagination, idempotency, admin-only writes.
  2. `capacity_ru` 1..60, `capacity_kw` > 0; label unique per room; archiving a rack with active allocations -> 409.
  3. Authorization tests per route including cross-tenant -> 404.
  4. Writes record `audit_log` entries.

#### CP-23: Reservation workflow (request -> approve -> allocate)
- Owner: builder | Risk: risk:medium | Size: ~780 lines | Depends on: CP-12, CP-16, CP-17, CP-18
- Files: as in overlap check
- Acceptance:
  1. State machine allows only the transitions in spec section 2; others -> 409 `invalid-transition`; each transition writes `reservation_events` and an outbox event in the same transaction.
  2. Approve and allocate lock the room row and call `check_fits`; 20 concurrent approvals of 10 kW into a room with 100 kW headroom -> exactly **10** succeed, 10 get 409 `capacity-exceeded`.
  3. Room-level allocation auto-picks racks with enough free kW and RU (fewest racks first) or accepts an explicit rack list; rack-level allocates that rack only.
  4. Approvers cannot approve their own request (PROPOSED, configurable); requesters can cancel only their own.
  5. Expiry schedule moves `approved` reservations not allocated within **14 days** (PROPOSED, configurable) to `expired`, in batches of 500.
  6. `events_notifier` consumer logs a structured "notification" per event using `inbox.once` (stand-in for email later).
  7. Authorization tests per route and role including cross-tenant -> 404.

#### CP-24: Forecast read service and API
- Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-12, CP-13
- Files: as in overlap check
- Acceptance:
  1. `GET /v1/sites/{site_id}/forecasts/latest?metric=` returns 90 points with intervals, model name, run id and `generated_at`; 404 if none; `status=insufficient_history` returned as 200 with empty points.
  2. Read p95 <= **200 ms** on seeded data (micro-benchmark in PR).
  3. Service function `latest_forecast(ctx, site_id, metric, horizon_days<=90)` is reusable by the agent tools.
  4. Authorization tests including cross-tenant -> 404.

#### CP-25: Seed telemetry history and synthetic policy documents
- Owner: data-engineer | Risk: risk:medium | Size: ~600 lines (documents count as content) | Depends on: CP-13, CP-14
- Files: as in overlap check
- Acceptance:
  1. Loads 730 days of daily and 30 days of hourly rollups for every seeded room, and 2 days of raw 60-second points, deterministic with `SEED=42`, idempotent on re-run.
  2. Writes the synthetic policy documents agreed in Q8 (default set: power loading policy, reservation and approval policy, rack standards, change-freeze calendar, capacity planning guideline, cooling guideline), each 1,500-4,000 words, clearly marked synthetic, with facts the golden set can reference.
  3. Data quality tests: freshness of latest raw point, no gaps beyond the planned 1%, bad rows flagged not loaded.
  4. Loading completes in under **5 min** on the reference laptop.

#### CP-26: Rate limits, secure headers, CORS, body size limits
- Owner: security | Risk: risk:high (security) | Size: ~550 lines | Depends on: CP-06, CP-12
- Files: as in overlap check
- Acceptance:
  1. Rate limit per tenant (**50 requests/second**, burst 100) and per IP for unauthenticated requests (**20 requests/second**) using Redis; exceed -> 429 + `Retry-After`; `test_rate_limit_per_tenant` passes.
  2. `test_security_headers_present`: Content-Security-Policy, Strict-Transport-Security, X-Content-Type-Options, frame-ancestors 'none', Referrer-Policy.
  3. CORS allow-list contains only `http://localhost:18080` by default (configurable); other origins rejected.
  4. Bodies over **1 MB** (2 MB for document upload) -> 413 before parsing.
  5. Threat model updated for all Wave 4 endpoints.

#### CP-27: Document upload, chunking and embedding pipeline
- Owner: builder | Risk: risk:high (queues, external AI call) | Size: ~750 lines | Depends on: CP-09, CP-12, CP-13, CP-16, CP-17; real adapter needs Q3
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/documents` accepts `.md`/`.txt` up to 2 MB, dedupes by SHA-256, stores the document `pending` and an outbox event to `cp:documents` (cap **500**, 429 when full) in one transaction.
  2. Consumer chunks to about **500 tokens** with **50-token** overlap, embeds in batches through the resilience client (timeouts, retries), writes chunks and tsv, marks `embedded`; failures retry then dead-letter and mark `failed`.
  3. `Embedder` interface with a deterministic `fake` provider (tests) and the chosen vendor's adapter; vector size checked against the column (1,024 until Q3).
  4. A 2 MB document is searchable within **120 s** locally; delete removes chunks.
  5. Authorization tests: only admin uploads/deletes; cross-tenant -> 404.

### Wave 5: forecast job, capacity endpoints, retrieval, guards, golden data, first screens

Overlap check: CP-28 = `forecast/src/cp_forecast/job.py`, `worker/src/cp_worker/schedules/nightly_forecast.py`, `api/src/cp_api/routers/forecast_runs.py`, `api/src/cp_api/schemas/forecast_runs.py`, `contracts/openapi/{paths,components}/forecast-runs.yaml`, `forecast/tests/job/**`, `api/tests/forecast_runs/**`;
CP-29 = `api/src/cp_api/routers/{capacity,utilization}.py`, `api/src/cp_api/schemas/capacity.py`, `contracts/openapi/paths/{capacity,utilization}.yaml`, `contracts/openapi/components/capacity.yaml`, `api/tests/capacity/**`;
CP-30 = `agent/src/cp_agent/retrieval.py`, `agent/tests/retrieval/**`;
CP-31 = `seed/src/cp_seed/load_documents.py`, `seed/tests/documents/**`;
CP-32 = `agent/src/cp_agent/guards.py`, `agent/tests/guards/**`, `docs/threat-model.md`;
CP-33 = `evals/golden/**`;
CP-34 = `tests/integration/**`;
CP-35 = `apps/web/lib/api/**`, `apps/web/app/sites/**`, `apps/web/e2e/sites.spec.ts`. No shared files.

#### CP-28: Nightly forecast job, schedule and manual trigger
- Owner: analytics | Risk: risk:medium | Size: ~600 lines | Depends on: CP-10, CP-16, CP-20, CP-24
- Files: as in overlap check
- Acceptance:
  1. Job loops over tenants (as `cp_scheduler`), then sites per tenant (as `cp_app` with tenant set), fits `peak_kw` and `ru_used`, writes `forecasts` and `forecast_points`; one site's failure marks it `failed` and the run `partial`, others continue.
  2. Scheduled at **02:00 UTC**; finishes **500 sites x 2 metrics** in under **30 min** (test with synthetic sites; PR shows timing).
  3. `POST /v1/forecast-runs` (admin) enqueues a run (202, idempotent); `GET /v1/forecast-runs` paginated.
  4. Re-running the same day replaces nothing silently: each run is a new `forecast_runs` row and "latest" points to the newest successful forecast.
  5. Plain-English interpretation of backtest results on seeded data in the PR.

#### CP-29: Capacity summary and utilization endpoints
- Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-18, CP-20, CP-21
- Files: as in overlap check
- Acceptance:
  1. `GET /v1/sites/{site_id}/capacity` returns per-room and site totals of capacity, committed, used and headroom for kW and RU using only `capacity.py` and `utilization.py`.
  2. `GET /v1/sites/{site_id}/utilization` supports `metric`, `grain=hour|day`, `from`, `to` (max 400 days), keyset pagination on `ts`.
  3. Both meet read p95 <= **200 ms** on seeded data (micro-benchmark in PR).
  4. Authorization tests including cross-tenant -> 404.

#### CP-30: Hybrid retrieval over policy chunks
- Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-27
- Files: as in overlap check
- Acceptance:
  1. `search(ctx, query, k<=8)` runs pgvector cosine top 20 and full-text top 20 under tenant RLS, merges with reciprocal rank fusion, returns top k with document title, chunk id and text.
  2. Enables pgvector iterative index scan so tenant filtering still returns k results when they exist (tested with 3 tenants).
  3. Query embedding call has a timeout of **2 s** and one retry; p95 search latency <= **300 ms** locally (excluding embedding vendor time, which is reported separately).
  4. Unit tests with the fake embedder; recall@5 measured later by `make eval`.

#### CP-31: Load synthetic policy documents through the API
- Owner: data-engineer | Risk: risk:medium | Size: ~200 lines | Depends on: CP-25, CP-27
- Files: as in overlap check
- Acceptance:
  1. `make seed` uploads every file in `seed/policies/` for each tenant via `POST /v1/documents` with idempotency keys, then waits until all are `embedded` (timeout **5 min**).
  2. Re-running uploads nothing new (dedupe by hash).
  3. Test verifies chunk counts are greater than 0 for every document.

#### CP-32: AI guards: untrusted-content wrapping, tool argument validation, output checks
- Owner: security | Risk: risk:high (AI agent loop, security) | Size: ~400 lines | Depends on: CP-12
- Files: as in overlap check
- Acceptance:
  1. `wrap_untrusted(source, text)` places retrieved text and tool results inside marked data blocks and strips control sequences that mimic system or tool markers.
  2. `validate_tool_call(name, args)` rejects any tool not on the allow-list and any argument failing its schema (including any attempt to pass `tenant_id`).
  3. `check_output(answer)` flags system-prompt leakage and secret-like strings; flagged answers are replaced by a safe refusal.
  4. `test_untrusted_content_is_wrapped_and_tool_args_validated` passes; threat model updated for the planner endpoint and tools.

#### CP-33: Golden evaluation datasets
- Owner: qa | Risk: risk:low (tests only) | Size: ~700 lines (data) | Depends on: CP-25
- Files: `evals/golden/{retrieval,planner_qa,out_of_scope,injection,human_labels}.jsonl`, `evals/golden/README.md`
- Acceptance:
  1. Sizes as in spec section 6: 100 retrieval, 100 in-scope (40 policy, 30 live-data, 30 mixed), 40 out-of-scope, 20 injection, 30 human-label slots.
  2. Every live-data reference answer is computed from the `SEED=42` dataset with the query used to compute it recorded.
  3. Every retrieval item lists expected chunk ids by `(document filename, heading)` so ids survive re-chunking.
  4. A schema test validates every line; no real company or customer data.

#### CP-34: Integration tests for tenancy, pagination and authorization across all endpoints
- Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-21, CP-22, CP-23, CP-24, CP-27
- Files: `tests/integration/**`, `reports/standards/<date>.md`
- Acceptance:
  1. `test_cross_tenant_access_returns_404_for_every_endpoint` iterates every route in the bundled contract.
  2. `test_every_list_route_is_paginated` checks every list route rejects `limit=101` and returns stable pages.
  3. Writes a target-vs-actual table with pass/fail per standard to `reports/standards/`.
  4. Any bug found is filed as an issue, not fixed.

#### CP-35: Web typed API client and Sites screens
- Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-11, CP-21, CP-22
- Files: as in overlap check
- Acceptance:
  1. Generated types from the bundled contract; a thin client adds auth header, `Idempotency-Key` on writes and handles problem+json and 429 `Retry-After`.
  2. Sites list (cursor Next/Previous), site detail with rooms, room detail with racks; admin can create/edit/archive.
  3. Playwright test covers list, paging, create and an error message for 409.
  4. Pages render in under **1 s** locally on seeded data.

### Wave 6: agent tools, eval harness, more screens, load tests, CI hardening, data maintenance

Overlap check: CP-36 = `agent/src/cp_agent/tools/**`, `agent/tests/tools/**`;
CP-37 = `evals/src/cp_evals/**`, `evals/config/**`, `evals/tests/**`;
CP-38 = `apps/web/app/reservations/**`, `apps/web/e2e/reservations.spec.ts`;
CP-39 = `apps/web/app/forecasts/**`, `apps/web/e2e/forecasts.spec.ts`;
CP-40 = `load/**`, `reports/load/**`;
CP-41 = `.github/workflows/ci.yml`, `.github/dependabot.yml`, `infra/scripts/check-actions-pinned.sh`;
CP-42 = `worker/src/cp_worker/schedules/{partition_maintenance,retention}.py`, `worker/src/cp_worker/jobs/backfill_rollups.py`, `core/src/cp_core/jobs/{__init__,checkpoints}.py`, `worker/tests/maintenance/**`. No shared files.

#### CP-36: Planner agent tools (read-only, tenant-scoped)
- Owner: builder | Risk: risk:high (AI agent loop) | Size: ~600 lines | Depends on: CP-24, CP-29, CP-30, CP-32
- Files: as in overlap check
- Acceptance:
  1. Tools `search_policies`, `get_site_capacity`, `get_utilization`, `get_forecast`, `list_reservations` with Pydantic argument schemas and JSON-schema definitions for Claude; none accepts `tenant_id`.
  2. Each tool calls existing services only (no new SQL), runs under the caller's tenant, returns at most **4,000 characters** wrapped by `wrap_untrusted`.
  3. `test_agent_tools_cannot_read_other_tenant` passes for every tool.
  4. Each tool has a **3 s** timeout and returns a structured error result instead of raising.

#### CP-37: Evaluation harness and `make eval`
- Owner: qa | Risk: risk:low (tests only) | Size: ~700 lines | Depends on: CP-30, CP-33
- Files: as in overlap check
- Acceptance:
  1. `make eval` seeds `cp_eval` with `SEED=42`, runs retrieval metrics now and planner metrics when `cp_agent.answer` exists, and prints a table of every target in spec section 6 with actual and pass/fail.
  2. Reports p95 time to first token, p95 total time, and mean and p95 cost per question in dollars from a dated price table in `evals/config/prices.yaml`; judge cost reported separately.
  3. Judge calls use Claude at temperature 0 with `max_tokens` 512; judge agreement with `human_labels.jsonl` reported.
  4. Writes `reports/eval/<timestamp>.md` and `.json`; exits non-zero if any gated target fails; `test_make_eval_reports_quality_latency_cost` passes.
  5. `make eval EVAL_SUBSET=smoke` runs 20 questions for under **$1**.

#### CP-38: Web reservation screens
- Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-23, CP-35
- Files: as in overlap check
- Acceptance:
  1. Request form (site, room, optional rack, kW, RU, dates, justification) with client-side validation matching the API.
  2. List filtered by status with cursor paging; detail shows history and allocations.
  3. Approver view with approve, reject, allocate (auto or choose racks), release; buttons shown only for allowed roles.
  4. `capacity-exceeded` and `invalid-transition` errors shown in plain English; Playwright covers the full request -> approve -> allocate path.

#### CP-39: Web capacity and forecast screens
- Owner: builder | Risk: risk:medium | Size: ~600 lines | Depends on: CP-24, CP-28, CP-29, CP-35
- Files: as in overlap check
- Acceptance:
  1. Site capacity table per room (capacity, committed, used, headroom for kW and RU) with headroom under **20%** highlighted (PROPOSED threshold).
  2. 90-day forecast chart with 80% and 95% bands, committed reservations overlaid, and capacity line.
  3. Each chart and number names its metric from docs/metrics.md in a tooltip.
  4. Playwright test checks the chart renders for a seeded site and shows "Not enough history" for a new site.

#### CP-40: Load tests for API SLOs and ingest
- Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-20, CP-21, CP-22, CP-23, CP-24, CP-29
- Files: `load/**`, `reports/load/**`
- Acceptance:
  1. `make load` runs the API mix at **50 requests/second** for 10 min (1 min warm-up) and reports read p95, write p95 and 5xx rate vs targets.
  2. Ingest scenario at **2,000 points/second** for 15 min reports throughput, 202 p95, freshness p95/p99, 429 count, and loss = accepted - stored - dead-lettered.
  3. Burst scenario at **4,000 points/second** for 2 min proves 429s occur, the cap is never exceeded and the queue drains below 5,000 within 60 s.
  4. Each scenario is run 3 times and the median reported in `reports/load/` with pass/fail per row.

#### CP-41: CI hardening
- Owner: release | Risk: risk:high (CI/CD) | Size: ~300 lines | Depends on: CP-15
- Files: `.github/workflows/ci.yml`, `.github/dependabot.yml`, `infra/scripts/check-actions-pinned.sh`
- Acceptance:
  1. CI installs uv and pnpm (pinned, checksum-verified), runs `make check` and `make test` with Testcontainers, and builds images for arm64 and amd64.
  2. `check-actions-pinned.sh` fails the build if any action is not pinned to a 40-character SHA.
  3. CI generates an SBOM and runs Trivy on built images (zero unfixed HIGH/CRITICAL).
  4. Dependabot enabled for pip/uv, npm, Docker and GitHub Actions; every job has `timeout-minutes`.

#### CP-42: Partition maintenance, retention and batched rollup backfill
- Owner: data-engineer | Risk: risk:high (deleting data) | Size: ~450 lines | Depends on: CP-16, CP-20
- Files: as in overlap check
- Acceptance:
  1. Daily job creates telemetry partitions **14 days** ahead; retention job drops raw partitions older than the approved retention (14 days PROPOSED, Q6) and deletes rollups past their retention in batches of 5,000.
  2. `backfill_rollups` recomputes rollups for a date range in keyset batches of **5,000** rows, checkpointing in `job_checkpoints`; `test_backfill_is_batched_and_resumable` passes.
  3. Retention never drops a partition for today or the future (tested); dry-run mode lists what would be dropped.
  4. Each job logs rows affected and duration; PR shows `EXPLAIN` for the batch query.

### Wave 7: agent loop, real auth (blocked), documents screen

Overlap check: CP-43 = `agent/src/cp_agent/{loop,llm,registry,answer}.py`, `agent/src/cp_agent/prompts/system.md`, `agent/tests/loop/**`;
CP-44 = `api/src/cp_api/auth/providers/**`, `api/tests/auth/providers/**`, `docs/threat-model.md` (and `uv.lock` if a library is needed; it is the only W7 ticket allowed to touch it);
CP-46 = `apps/web/app/documents/**`, `apps/web/e2e/documents.spec.ts`. No shared files.
(CP-45 is numbered here but scheduled in Wave 8 because it depends on CP-43.)

#### CP-43: Planner agent loop with caps, cost guard and streaming
- Owner: builder | Risk: risk:high (AI agent loop) | Size: ~700 lines | Depends on: CP-09, CP-32, CP-36, CP-37
- Files: as in overlap check
- Acceptance:
  1. `answer(ctx, question)` streams events (`tool`, `token`, `citation`, `done`) using Claude via the resilience client; model from `CP_AGENT_MODEL`; system prompt and tool definitions use prompt caching.
  2. `registry.py` is the explicit tool allow-list; `test_agent_stops_after_8_tool_calls`, `test_agent_hard_timeout_15s`, `test_every_llm_call_sets_max_tokens` (1,024 final / 512 intermediate) and `test_cost_guard_stops_before_limit` pass.
  3. Refuses out-of-scope questions with a fixed polite message; answers cite document chunks and tool results.
  4. `test_trace_continues_into_llm_call` passes; spans record tokens and cost in dollars; prompts with user data are not logged.
  5. PR includes a full `make eval` table meeting every gated target in spec section 6, or states the gap and stops for director review.

#### CP-44: Real authentication provider (BLOCKED on Q1)
- Owner: security | Risk: risk:high (auth, dependencies) | Size: ~600 lines | Depends on: CP-12; director answer to Q1
- Files: as in overlap check
- Acceptance:
  1. Implements `AuthProvider` for the chosen provider (token validation, signature keys rotation, expiry, audience, issuer).
  2. Maps provider identities and groups to `tenant_id` and the roles in spec section 2.
  3. Failed logins back off per user and IP (security rule); logins and permission changes audit-logged.
  4. All existing authorization tests pass unchanged with the new provider in test mode; threat model updated.

#### CP-45: Failure-injection tests (scheduled in Wave 8)
- Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-20, CP-23, CP-27, CP-43
- Files: `tests/failure/**`, `reports/failure/**`
- Acceptance:
  1. Redis stopped during ingest -> 503 + `Retry-After`, no 5xx storm, recovery within **30 s** of Redis returning.
  2. Worker killed mid-batch -> items reclaimed after 60 s; no loss, no duplicates.
  3. Postgres connection drop during reservation approval -> no half-applied state (event, reservation and outbox agree).
  4. LLM vendor returns 429/500 or hangs -> retries with backoff, then a graceful error event within 15 s.
  5. Results table with pass/fail per scenario in `reports/failure/`.

#### CP-46: Web documents screen
- Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-27, CP-35
- Files: as in overlap check
- Acceptance:
  1. Admin uploads `.md`/`.txt` up to 2 MB, sees status move from pending to embedded or failed.
  2. List with cursor paging; delete with confirmation.
  3. Non-admins see the list read-only.
  4. Playwright covers upload, status refresh and delete.

### Wave 8: planner API, failure tests, nightly eval, staging (blocked)

Overlap check: CP-45 = `tests/failure/**`, `reports/failure/**` (defined under Wave 7 above);
CP-47 = `api/src/cp_api/routers/planner.py`, `api/src/cp_api/schemas/planner.py`, `core/src/cp_core/services/planner_history.py`, `contracts/openapi/{paths,components}/planner.yaml`, `api/tests/planner/**`;
CP-48 = `.github/workflows/nightly-eval.yml`;
CP-49 = `.github/workflows/deploy-staging.yml`, `infra/staging/**`. No shared files.

#### CP-47: Planner question API (Server-Sent Events) and history
- Owner: builder | Risk: risk:high (AI agent loop) | Size: ~550 lines | Depends on: CP-43
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/planner/questions` streams SSE events from `answer(...)`; first event `meta` includes `question_id`.
  2. Concurrency bounded at **20** in-flight questions per API instance and **5** per tenant; over the limit -> 429 + `Retry-After`; `test_planner_returns_429_when_concurrency_limit_reached` passes.
  3. Per-user limit **10 questions/minute** and per-tenant daily AI spend cap (**$10/day** PROPOSED, Q9) -> 429 when exceeded.
  4. Each question stored in `planner_questions` with status, citations, tool calls, tokens, cost in dollars and latency; list and detail endpoints paginated and tenant-scoped.
  5. End-to-end p95 time to first token <= **1.5 s** and total <= **6 s** measured through the API on the smoke eval subset.
  6. Authorization tests per route including cross-tenant -> 404 and `ingestor` -> 403.

#### CP-48: Nightly evaluation workflow
- Owner: release | Risk: risk:high (CI/CD) | Size: ~120 lines | Depends on: CP-37, CP-41, CP-43
- Files: `.github/workflows/nightly-eval.yml`
- Acceptance:
  1. Runs `make eval` nightly at **03:00 UTC** and on manual dispatch, with `ANTHROPIC_API_KEY` and embedding key from a GitHub environment secret.
  2. Uploads `reports/eval/` as an artifact; fails the job if any gated target fails.
  3. `timeout-minutes: 60`; actions pinned to SHAs; spend per run under **$10** (from the report).

#### CP-49: Staging deployment (BLOCKED on Q7)
- Owner: release | Risk: risk:high (infra, CI/CD) | Size: ~500 lines | Depends on: CP-41, CP-44; director answer to Q7
- Files: as in overlap check
- Acceptance:
  1. Deploys images to the chosen staging host only after director approval via the GitHub `staging` environment.
  2. Secrets only from GitHub environments; no credentials printed.
  3. Runs migrations as `cp_owner`, then smoke tests `/readyz` and one authenticated read.
  4. Rollback to the previous image documented and tested once.

### Wave 9: planner screen and final acceptance

Overlap check: CP-50 = `apps/web/app/planner/**`, `apps/web/e2e/planner.spec.ts`;
CP-51 = `tests/acceptance/**`, `reports/acceptance/**`;
CP-52 = no repository files (opens issues only). No shared files.

#### CP-50: Web planner chat screen
- Owner: builder | Risk: risk:medium | Size: ~500 lines | Depends on: CP-47
- Files: as in overlap check
- Acceptance:
  1. Chat input (<= 2,000 characters) streams the answer token by token, shows tool steps ("Checking DAL-1 forecast...") and clickable citations.
  2. Shows refusals, time-outs and 429 "busy, try again in N s" in plain English.
  3. History list of past questions with cursor paging.
  4. Playwright test with a mocked SSE stream covers streaming, citation click and error display.

#### CP-51: Final acceptance report
- Owner: qa | Risk: risk:low (tests only) | Size: ~300 lines | Depends on: all Wave 1-8 tickets except blocked ones
- Files: `tests/acceptance/**`, `reports/acceptance/**`
- Acceptance:
  1. One table with every target in spec sections 4, 5 and 6 and every named test in section 7: target, actual, pass/fail.
  2. Runs on a clean `make down && make up && make seed` from scratch.
  3. Lists every open issue labelled `found-by-agent` with severity.

#### CP-52: Pre-launch security audit (OWASP ASVS Level 1)
- Owner: security | Risk: risk:high (security agent rule) | Size: 0 lines of code | Depends on: CP-47
- Files: none (opens GitHub issues only)
- Acceptance:
  1. Checks the repository against ASVS Level 1 and docs/threat-model.md.
  2. Opens one issue per gap with `bug`, `s1|s2|s3` and risk labels; "S1 FOUND" reported to the director immediately if any.
  3. Summary of gaps by severity returned to the director in plain English.

### Ticket summary

| Wave | Tickets | Owners |
|---|---|---|
| 1 | CP-01 to CP-04 | release, builder, security, analytics |
| 2 | CP-05 to CP-11 | data-engineer, builder x5, analytics |
| 3 | CP-12 to CP-18 | security, data-engineer x2, release, builder x2, analytics |
| 4 | CP-19 to CP-27 | builder x6, data-engineer x2, security |
| 5 | CP-28 to CP-35 | analytics, builder x3, data-engineer, security, qa x2 |
| 6 | CP-36 to CP-42 | builder x3, qa x2, release, data-engineer |
| 7 | CP-43, CP-44, CP-46 | builder x2, security (blocked) |
| 8 | CP-45, CP-47 to CP-49 | qa, builder, release x2 (one blocked) |
| 9 | CP-50 to CP-52 | builder, qa, security |

---

## 10. Assumptions made by the architect

- Users and roles are placeholders until auth is chosen; `actor_id` is a string from the auth layer.
- Policy documents are Markdown or plain text only in v1 (PDF would add a parsing dependency).
- Embedding vector size is assumed to be **1,024** until the vendor is chosen.
- Local tracing uses an in-memory exporter in tests and an optional OTLP endpoint; no trace viewer is
  added because only four host ports are allowed.
- The 2,000 points/second ingest target is headroom; the synthetic steady state is about 85 points/second.
- The web app talks to the API with a dev token locally until real auth exists.
- The release agent may only edit `.github/`, `infra/` and Dockerfiles, so the Makefile is owned by the
  builder (CP-02) and calls `infra/` scripts.

---

## 11. Open questions for the director

Please answer each with a choice or "use the proposal". Items marked **blocks** stop the named tickets.

1. **Q1. Login provider (blocks CP-44, CP-49).** Which identity provider should we use (for example
   Auth0, Okta, Microsoft Entra ID, Keycloak self-hosted, Clerk)? Until then we use the local-only fake
   login (ADR 0008).
2. **Q2. Which Claude model?** A fast, low-cost Claude model is the safest way to meet **$0.02 per
   question** and **1.5 s** to first token. A larger model may answer better but could break the cost
   and speed targets. Proposal: start with the fast model, compare a larger one in `make eval`.
3. **Q3. Which embedding vendor and model (needed before Wave 4, CP-27)?** Proposal: Voyage AI (the
   embedding partner Anthropic recommends) with a 1,024-dimension model. This fixes the database
   vector size.
4. **Q4. Forecast accuracy targets.** Approve the PROPOSED targets in section 6 (kW WAPE <= 8% for
   1-30 days, <= 12% for 31-90 days; RU <= 5% / <= 8%; interval coverage bands; beat baseline by >= 15%)?
5. **Q5. Reservation approvals.** Who may approve (proposal: `approver` and `admin` roles, per tenant)?
   May someone approve their own request (proposal: no)? How long may an approved reservation wait
   before it expires unallocated (proposal: **14 days**)? Should site-level approvers exist (only
   approve for their own sites)?
6. **Q6. Raw telemetry retention.** Proposal: raw readings **14 days**, hourly summaries **400 days**,
   daily summaries **5 years**. Longer raw retention needs much more disk (about 173 million rows/day at
   full ingest rate).
7. **Q7. Staging hosting target (blocks CP-49).** Where should staging run (for example a cloud
   account, a managed container service, or the home K3s cluster)? This also sets the availability SLO.
8. **Q8. Synthetic policy documents to seed (needed for CP-25 and CP-33).** Proposal: power loading
   policy (rooms max 80% of rated kW), reservation and approval policy, rack standards (42 RU, 17 kW
   standard / 30 kW high-density), change-freeze calendar, capacity planning guideline (warn under 20%
   headroom), cooling guideline. Add or remove any?
9. **Q9. AI spending limits.** Approve a per-tenant cap of **$10/day** for the assistant and a nightly
   eval budget of **under $10/run**?
10. **Q10. Human labels for the AI judge.** Who labels the 30 calibration answers (about 1 hour of
    work)? Proposal: the director, using a simple form QA provides.
11. **Q11. Approve the PROPOSED SLOs** for forecast freshness (done by 02:30 UTC), document embedding
    (searchable within 120 s), and the 20% low-headroom highlight.
