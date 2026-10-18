# CapacityPilot specification

- Status: **Draft for director approval.** The director answered open questions Q1 to Q14 on
  2026-10-17, and ADRs 0001 to 0008 are Accepted; those answers are recorded below and in section 11.
  This spec becomes **approved when the director merges this pull request**. Nothing is built until then.
- Author: architect agent
- Date: 2026-10-17
- Decision records: docs/adr/0001 to 0008

How to read this document: every section starts with **Decision** in plain English. The detail below
it is for the agents building the system. Words in **bold** are defined on first use. Every target is
a number with a unit. Targets the director approved on 2026-10-17 are marked **(approved Qn)** so you
can trace them back to your answer.

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
deployment, email/SMS notifications (events are recorded so they can be added later), and a staging
environment (deferred to Phase 2, see Q7).

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
| Policy documents (approved Q8) | 6 synthetic documents per tenant: power loading policy (rooms max 80% of rated kW), reservation and approval policy, rack standards (42 RU; 17 kW standard / 30 kW high-density), change-freeze calendar, capacity planning guideline (warn under 20% headroom), cooling guideline |

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
    |--reject--> rejected |    \--(not allocated within 14 days, approved Q5)--> expired
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
  representing meaning, made by Voyage AI) for search.
- A **planner question** records each question to the assistant, the route it took, its answer,
  citations, tools used, time taken and cost.
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
| `document_chunks` [T] | id, document_id, ordinal, text, token_count, tsv (full-text vector), embedding `vector(1024)` | HNSW cosine index on embedding; GIN index on tsv. 1,024 dimensions matches the chosen Voyage AI model (Q3) |
| `planner_questions` [T] | id, actor_id, question (<= 2,000 chars), route (`out_of_scope`, `simple_lookup`, `full`), models_used, answer, status (`answered`, `refused`, `timed_out`, `budget_stopped`, `error`), citations (json), tool_calls (json), input_tokens, output_tokens (per model, json), cost_usd, ttft_ms, total_ms, created_at | One row per question |
| `outbox_events` [T] | id, stream, event_type, aggregate_type, aggregate_id, payload (json), traceparent, created_at, published_at, attempts | Written in the same transaction as the business change |
| `processed_events` [T] | event_id, consumer, processed_at | Unique `(consumer, event_id)` so repeats are skipped |
| `idempotency_keys` [T] | user_id, key, method, path, request_sha256, status_code, response_body, created_at, expires_at (24 h) | Primary key `(tenant_id, user_id, key)`, so two users in the same tenant never share a key |
| `audit_log` [T] | id, actor_id, action, target_type, target_id, at, trace_id, detail (json, no secrets) | Append-only |
| `job_checkpoints` | job_name, tenant_id (nullable), cursor (json), updated_at | Resumable batch jobs (ADR 0004) |

Users are **not** stored yet: `actor_id` is the user id string supplied by the auth layer (ADR 0008).
A `users`/`memberships` table is added when a real login provider is chosen (CP-44, Phase 2), in
migration `0003_users_memberships.py`, which is the only migration in Phase 2 wave P2 (ADR 0004 rule 7).

### Roles and permissions (approved Q5; login is the local dev provider for now, Q1)

Approvers act for their **whole tenant** (no site-level approvers in this version).

| Action | viewer | requester | approver | admin | ingestor |
|---|---|---|---|---|---|
| Read sites, rooms, racks, capacity, utilization, forecasts, reservations | yes | yes | yes | yes | no |
| Create, edit, archive sites, rooms, racks | no | no | no | yes | no |
| Send telemetry | no | no | no | no | yes |
| Create a reservation; cancel own reservation | no | yes | yes | yes | no |
| Approve, reject, allocate, release | no | no | yes (never their own request) | yes (never their own request) | no |
| Upload or delete policy documents | no | no | no | yes | no |
| Ask the planner assistant | yes | yes | yes | yes | no |
| Trigger a forecast run | no | no | no | yes | no |

---

## 3. API outline (OpenAPI 3.1)

**Decision:** One versioned web API under `/v1`, described by an OpenAPI 3.1 contract kept in
`contracts/openapi/` (split by resource, bundled automatically, ADR 0005). Every endpoint requires
login except two health checks. Lists use cursor pagination with at most 100 items per page (ADR 0002).
Every create or state-changing request accepts an `Idempotency-Key` so a retried click never happens
twice. Errors use one standard format. Endpoints marked **P2** arrive in Phase 2 (section 9).

### Conventions (apply to every endpoint)
- Base URL locally: `http://localhost:18000/v1`. Web app (Phase 2): `http://localhost:18080`.
- **Auth**: `Authorization: Bearer <token>`; the auth layer yields `tenant_id`, `user_id`, `roles`.
  Missing/invalid token, or a token that yields no tenant -> **401** (the caller is not properly
  logged in, so we cannot know whose data they may see; 401 tells them to log in again).
  Wrong role -> **403**. Another tenant's object -> **404**.
- **Idempotency**: `Idempotency-Key` header (1-128 printable characters) is **required** on every
  `POST` and accepted on `PATCH`/`DELETE`. Keys are stored per tenant **and per user**
  (`(tenant_id, user_id, key)`; telemetry uses `(tenant_id, key)` because the sender is a machine
  credential, see CP-19). Same key + same body within 24 h returns the original response; same key +
  different body -> **422**; same key still in progress -> **409**.
  **Crash safety**: if the server crashes mid-request, its key would otherwise stay "in progress"
  forever. So an in-progress key older than the endpoint's hard timeout plus a **60 s** margin is
  treated as failed and the request may be retried (for the assistant: 15 s + 60 s = **75 s**).
  **Streaming assistant (`POST /v1/planner/questions`)**: a repeat while the first answer is still
  streaming (and younger than 75 s) -> **409** + `Retry-After: 2`; a repeat after it finished returns
  the stored answer as a plain JSON body (the same shape as `GET /v1/planner/questions/{id}`, status
  **200**, header `Idempotent-Replay: true`), with no new model calls and no new cost. Exception: if
  the first attempt ended `timed_out` or `error`, or its key expired as above, the repeat is **run
  again** as a fresh answer (new model calls, new cost, counted against the usual limits and caps).
- **Pagination**: `limit` (1-100, default 50), `cursor`, `sort` from a per-endpoint allow-list.
  Response `{items, next_cursor}`.
- **Errors**: RFC 9457 `application/problem+json` with `type`, `title`, `status`, `detail`,
  `trace_id`. No stack traces. Types: `validation`, `unauthorized`, `forbidden`, `not-found`,
  `conflict`, `capacity-exceeded`, `invalid-transition`, `invalid-cursor`, `idempotency-mismatch`,
  `queue-full`, `rate-limited`, `unavailable`, `payload-too-large`.
- **Busy signals**: **429** + `Retry-After` (seconds) when a queue is full or a rate limit is hit;
  **503** + `Retry-After` when a dependency (Redis) is down.
- **Tracing**: accepts and returns W3C `traceparent`; every response has `X-Trace-Id`.
- **Limits (approved Q11)**: request body <= **1 MB** (documents upload <= **2 MB**); rate limit
  **50 requests/second per tenant** (burst 100) and **20 requests/second per IP** for unauthenticated
  attempts. The general rate-limit and header plugin is CP-26 (Phase 2); the MVP enforces the planner
  limits in CP-47 and the ingest queue cap in CP-19.

### Endpoints

| Method and path | Purpose | Roles | SLO class |
|---|---|---|---|
| `GET /healthz` | Process is alive (public) | public | - |
| `GET /readyz` | Postgres and Redis reachable (public) | public | - |
| `GET /v1/sites` (P2) | List sites (sort: `code`, `created_at`) | read roles | read |
| `POST /v1/sites` (P2) | Create site | admin | write |
| `GET /v1/sites/{site_id}` (P2) | Get site | read roles | read |
| `PATCH /v1/sites/{site_id}` (P2) | Update site | admin | write |
| `DELETE /v1/sites/{site_id}` (P2) | Archive site (409 if active reservations) | admin | write |
| `GET /v1/sites/{site_id}/rooms` (P2) | List rooms (sort: `name`, `created_at`) | read roles | read |
| `POST /v1/sites/{site_id}/rooms` (P2) | Create room | admin | write |
| `GET, PATCH, DELETE /v1/rooms/{room_id}` (P2) | Get, update, archive room | read roles / admin | read / write |
| `GET /v1/rooms/{room_id}/racks` (P2) | List racks (sort: `label`, `created_at`) | read roles | read |
| `POST /v1/rooms/{room_id}/racks` (P2) | Create rack | admin | write |
| `GET, PATCH, DELETE /v1/racks/{rack_id}` (P2) | Get, update, archive rack | read roles / admin | read / write |
| `GET /v1/sites/{site_id}/capacity` (P2) | Per-room capacity, committed, used, headroom (kW and RU) | read roles | read |
| `GET /v1/sites/{site_id}/utilization` (P2) | Hourly or daily rollups, `metric`, `from`, `to`, `grain`; paginated (sort: `ts`) | read roles | read |
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
| `GET /v1/planner/questions/{id}` | Stored answer, route, citations, tool trace, cost | human roles | read |

At MVP, sites, rooms and racks come from the synthetic seed (CP-14); editing them through the API is
Phase 2 (CP-21, CP-22).

### Telemetry batch shape
Request: `{"points": [{"rack_id": uuid, "metric": "power_kw"|"ru_used", "ts": RFC 3339 UTC, "value": number}]}`,
1-500 points, body <= 256 KB. `ts` may be at most **5 minutes** in the future and **7 days** in the past.
Response **202**: `{"batch_id", "accepted": n, "rejected": n, "items": [{"index", "status": "accepted"|"rejected", "reason"}]}`.
The whole batch is refused with **429** (nothing enqueued) if accepting it would exceed the queue cap.
Body that fails schema validation entirely -> **422**.

---

## 4. Service level objectives (SLOs)

**Decision:** At **50 requests per second**, 95% of reads finish within **200 ms** and 95% of writes
within **400 ms**. Each endpoint ticket shows a micro-benchmark in its PR; the full automated load test
(CP-40) runs in Phase 2 on a defined laptop setup.

Plain English: **p95** means "95 out of 100 requests are at least this fast". An **SLO** is a target
we promise to meet and measure.

| SLO | Target | How measured |
|---|---|---|
| Read latency (GET) | p95 **<= 200 ms** at **50 requests/second** | Per-endpoint micro-benchmarks in MVP PRs; full load test CP-40 (Phase 2): 10 min steady state after 1 min warm-up, 80% reads / 20% writes |
| Write latency (POST, PATCH, DELETE) | p95 **<= 400 ms** at **50 requests/second** | Same |
| Server error rate | **<= 0.1%** of requests return 5xx (429 excluded) | Same |
| Planner assistant | see section 6 | `make eval` |
| Ingest | see section 5 | CP-20 integration test (MVP); CP-40 load test (Phase 2) |
| Forecast freshness (approved Q11) | Nightly run starts **02:00 UTC** and finishes within **30 min** (by **02:30 UTC**) for **500 sites x 2 metrics**; every eligible site has a forecast **<= 26 h** old | Job metrics, CP-28 test |
| Document embedding (approved Q11) | A 2 MB Markdown document is searchable within **120 s** of upload | CP-27 test |
| Availability | **Deferred** with staging to Phase 2 (Q7); will be set when a staging host is chosen | - |

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
| Retention (approved Q6) | raw points **14 days**; hourly rollups **400 days**; daily rollups **5 years** |

MVP proves the throughput, cap, 429, freshness and no-duplicate targets in CP-08, CP-19 and CP-20
tests. The 15-minute sustained run and the burst test are CP-40 (Phase 2). The retention job is CP-42
(Phase 2); until then a local MVP stack keeps raw data, which is fine for synthetic volumes.

---

## 6. AI evaluation targets (planner assistant)

**Decision:** The assistant must find the right policy passages, stick to the facts it was given, get
answers right, refuse off-topic requests, ignore planted instructions, start answering within
**1.5 seconds**, finish within **6 seconds**, and cost no more than **2 cents** per question. Each
question is first sorted by **Claude Haiku 5.5**: off-topic questions are refused, simple lookups are
answered by Haiku, and full planning questions are answered by **Claude Sonnet 5.5**. **Claude Opus
5.5** is only used inside `make eval` as a comparison, never for real requests (approved Q2,
ADR 0007). A `make eval` command checks all of this against a fixed set of test questions and prints
pass/fail.

### Rules (from CLAUDE.md, mandatory)
- Golden dataset lives in `evals/golden/`.
- `make eval` reports **quality**, **p95 latency** and **cost per request in dollars**, writes a
  target-vs-actual table to `reports/eval/`, and exits non-zero if any gated target fails.
- Agent loop: **max 8 tool calls** per question (all models together), **hard timeout 15 s**
  (approved Q11), `max_tokens` **always set**: 64 for the router, 512 for intermediate steps,
  1,024 for the final answer, 512 for judge calls.

### Models (approved Q2; embeddings approved Q3)

| Purpose | Model id | Setting | In live requests? |
|---|---|---|---|
| Router (classify the question) and simple-lookup answers | `claude-haiku-5-5` | `CP_AGENT_ROUTER_MODEL` | yes |
| Full answers | `claude-sonnet-5-5` | `CP_AGENT_ANSWER_MODEL` | yes |
| Comparison baseline in `make eval` | `claude-opus-5-5` | `CP_EVAL_COMPARE_MODEL` | **no, eval only** (start-up check refuses Opus in the live settings) |
| Eval judge (scores faithfulness and correctness) | `claude-sonnet-5-5` | `CP_EVAL_JUDGE_MODEL` | no, eval only |
| Embeddings | Voyage AI, 1,024 dimensions | `CP_EMBED_PROVIDER=voyage`, `CP_EMBED_MODEL` | yes |

Because the judge is the same model family that writes full answers, the judge is checked against 30
human-labelled answers (director labels them, approved Q10) to catch self-favouring scores.

### Why this fits in $0.02 per question
- Cost per question = sum over every model call of (input tokens x input price + cached tokens x
  cache price + output tokens x output price) + query embedding cost, using a **dated price table** in
  `evals/config/prices.yaml` filled from Anthropic's and Voyage AI's published prices when CP-37 is built.
- Budgets by path, enforced by the cost guard (ADR 0007):

| Path | What happens | Budget |
|---|---|---|
| Router | Haiku, `max_tokens` 64, cached system prompt | **<= $0.002** |
| `out_of_scope` | fixed refusal, no further model calls | **<= $0.002** total |
| `simple_lookup` | Haiku runs up to 3 tool calls and answers | **<= $0.008** total |
| `full` | Sonnet runs up to 8 tool calls (minus any already used) and answers | **<= $0.02** total |

- Before every model call the cost guard adds up spend so far and the worst case of the next call (at
  its `max_tokens`). If that would pass **$0.02**, the loop stops and answers with what it has
  (`budget_stopped`). So no live question can exceed $0.02 by construction; the eval checks that
  `budget_stopped` is rare (see "budget stops" below).
- Levers if the full path runs over: prompt caching for both models' system prompts and tool
  definitions, tool results capped at 4,000 characters, parallel tool calls in one turn, and, if still
  needed, a "Haiku gathers, Sonnet writes once" mode (new ticket).

### Targets (all approved)

| Metric | Plain-English meaning | Target | Gated |
|---|---|---|---|
| Retrieval recall@5 | Share of the expected policy passages that appear in the top 5 search results, averaged over questions | **>= 0.85** | yes |
| Faithfulness | Share of statements in answers that are supported by the retrieved text or tool results (judge-scored) | **>= 0.90** | yes |
| Answer correctness | Share of in-scope questions answered correctly vs the reference; numbers must be within **2%** | **>= 0.80** | yes |
| Out-of-scope refusal | Share of off-topic questions politely refused | **>= 0.95** | yes |
| False refusal (approved Q11) | Share of in-scope questions wrongly refused | **<= 0.05** | yes |
| Prompt-injection resistance | Share of injection cases where the planted instruction is ignored | **= 1.00** (security rule) | yes |
| Citation presence (approved Q11) | Share of in-scope answers with at least one citation | **>= 0.95** | yes |
| Time to first answer token | From API receiving the question (including the router step) to the first streamed answer token, p95 | **<= 1.5 s** | yes |
| Total time | From question received to `done` event, p95 | **<= 6 s** | yes |
| Cost per question | All live model calls + query embeddings, p95, in dollars | **<= $0.02** | yes |
| Budget stops | Share of in-scope questions that ended `budget_stopped` | **<= 0.05** | yes |
| Tool-call cap | Share of runs with 8 or fewer tool calls | **= 1.00** | yes |
| Hard timeout | Share of runs ending within 15 s | **= 1.00** | yes |
| Opus never in live path | Live runs whose `models_used` includes an Opus model | **= 0** | yes |
| Routing accuracy | Share of questions whose route matches the golden label | **>= 0.90** | report only |
| Judge reliability (approved Q11) | Agreement of the AI judge with 30 human-labelled answers | **>= 0.85** | report only |
| Opus comparison | Correctness, faithfulness, p95 total time and cost per question for Opus vs the live Sonnet full path, on a fixed 40-question subset | difference reported in points and dollars | report only |

`make eval` reports every metric overall **and per route** (out-of-scope, simple lookup, full), with
cost per question in dollars split by model.

### Golden dataset (in `evals/golden/`)

| File | Contents | Size |
|---|---|---|
| `retrieval.jsonl` | question + ids of the passages that should be found | **100** questions |
| `planner_qa.jsonl` | in-scope questions with reference answers and expected route: 40 policy-only, 30 live-data, 30 mixed; at least 30 labelled `simple_lookup` | **100** questions |
| `out_of_scope.jsonl` | off-topic requests (recipes, legal advice, other tenants' data, code writing) | **40** questions |
| `injection.jsonl` | planted instructions: 10 inside policy documents, 10 inside user questions or data fields | **20** cases |
| `compare_subset.jsonl` | ids of 40 in-scope `full`-route questions used for the Opus comparison | **40** ids |
| `human_labels.jsonl` | 30 answers labelled by the director to calibrate the judge | **30** items |

Live-data answers are reproducible because `make eval` seeds a separate `cp_eval` database with
`SEED=42`.

### Eval spending (approved Q9, now including the Opus comparison)

One full `make eval` run is hard-capped at **$10.00**; the harness stops and fails the run if it would
go over. Planned split:

| Part | Questions | Budget |
|---|---|---|
| Live path (Haiku + Sonnet + embeddings) | 260 | **<= $5.20** (260 x $0.02 worst case) |
| Judge calls (Sonnet) | about 260 | **<= $1.50** |
| Opus comparison (eval only) | 40 | **<= $3.00** (about $0.075 per question) |
| Margin | - | **$0.30** |

`make eval EVAL_SUBSET=smoke` runs 20 questions without the Opus comparison for under **$1**. The
nightly CI eval (CP-48, Phase 2) runs the full set within the same **$10** cap. The live assistant
also has a per-tenant cap of **$10/day** (approved Q9).

### Forecast backtest targets (approved Q4)

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
breaks the rule, that test fails. Every standard is already proven by at least one **MVP** test; tests
from Phase 2 tickets (marked P2) add coverage later. QA writes a target-vs-actual table in `reports/`.

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
| Retries | `test_idempotency_key_scoped_per_tenant_and_user` | Same key from two tenants or two users never collides | CP-17 |
| Retries | `test_telemetry_idempotency_key_scoped_per_tenant` | Same telemetry key from two tenants: both batches stored, no shared reply | CP-19 |
| Retries | `test_planner_idempotent_replay_returns_stored_answer` | Repeat while streaming -> 409; repeat after `done` -> stored answer, no new model calls | CP-47 |
| Retries | `test_in_progress_idempotency_key_expires_after_timeout_and_can_retry` | A key left "in progress" by a crash is treated as failed after 75 s (15 s timeout + 60 s) and the retry runs; a retry after a `timed_out` or `error` attempt also runs again | CP-47 |
| Partial failure | `test_poison_message_moves_to_dead_letter_after_5_deliveries` | Bad item goes to DLQ with error and trace id; others continue | CP-08, CP-20 |
| Partial failure | `test_outbox_event_published_only_after_commit` | Rolled-back change publishes nothing; committed change publishes once | CP-16 |
| Partial failure | `test_outbox_relay_crash_does_not_lose_or_duplicate_effects` | Kill relay mid-batch; every event's effect happens exactly once | CP-16 |
| Partial failure | `test_batch_ingest_returns_per_item_status` | Mixed good/bad batch returns status per item | CP-19 |
| Pagination | `test_page_size_capped_at_100` | `limit=101` -> 422 | CP-07 |
| Pagination | `test_keyset_pagination_no_duplicates_or_gaps_under_concurrent_inserts` | Paging while inserting sees each original row exactly once | CP-07 |
| Pagination | `test_ties_broken_by_id` | Equal sort values come back in stable id order | CP-07 |
| Pagination | `test_tampered_cursor_rejected` | Edited cursor -> 400 | CP-07 |
| Pagination | `test_every_list_route_is_paginated` | Every list route has `limit`/`cursor` and enforces max 100 | CP-34 (P2) |
| Migrations | `test_all_migrations_upgrade_downgrade_roundtrip` | upgrade -> downgrade -> upgrade on seeded data | CP-05 (extended by CP-13) |
| Migrations | `test_single_alembic_head` | Exactly one migration head | CP-05 |
| Migrations | `test_no_destructive_ops_outside_contract_migrations` | No drops/renames in expand migrations | CP-05 |
| Migrations | `test_backfill_is_batched_and_resumable` | Backfill stops at batch k, restarts, finishes with no duplicates | CP-42 (P2) |
| Tenancy | `test_every_tenant_table_has_tenant_id_and_forced_rls` | Schema scan of every table | CP-05 |
| Tenancy | `test_rls_blocks_query_without_tenant_setting` | Query without `app.tenant_id` errors | CP-05 |
| Tenancy | `test_app_role_cannot_bypass_rls` | `cp_app` cannot see other tenants, even reading partitions | CP-05, CP-13 |
| Tenancy | `test_child_cannot_reference_other_tenant_parent` | Composite foreign keys hold | CP-05 |
| Tenancy | `test_app_role_cannot_create_or_drop_partitions_directly` | `cp_app` has no DDL rights; partitions change only through the `cp_owner` SECURITY DEFINER functions | CP-13 |
| Tenancy | `test_retention_function_refuses_cutoff_inside_14_days` | The drop function refuses any cutoff newer than today minus 14 days; `cp_app` cannot call it, `cp_scheduler` can | CP-13, CP-42 (P2) |
| Tenancy | `test_cross_tenant_access_returns_404_for_every_endpoint` | Each endpoint tried with another tenant's id | Per-endpoint tests in every MVP endpoint ticket; full sweep CP-34 (P2) |
| Tenancy | `test_agent_tools_cannot_read_other_tenant` | Tools under tenant A never return tenant B data | CP-36 |
| Observability | `test_logs_are_json_with_trace_id` | Every log line is JSON with `trace_id` | CP-06 |
| Observability | `test_trace_spans_api_to_queue_to_worker` | One trace id spans API, stream entry and worker span | CP-20 |
| Observability | `test_trace_continues_into_llm_call` | Planner trace includes router and answer LLM spans with token and cost attributes | CP-43 |
| Security | `test_every_route_requires_auth_unless_public` | Only `/healthz` and `/readyz` are public | CP-12 |
| Security | `test_every_route_has_authorization_test` | Every route appears in an authorization test, including a cross-tenant case | CP-12 |
| Security | `test_dev_auth_refuses_non_local_env` | Fake login cannot run outside local/test | CP-12 |
| Security | `test_security_headers_present` and `test_rate_limit_per_tenant` | Headers, CORS allow-list, 429 on abuse | CP-26 (P2) |
| Security | `check-stack.sh: db_and_redis_bound_to_localhost` and `check-stack.sh: redis_requires_auth` | Postgres and Redis listen on 127.0.0.1 only; Redis refuses commands without a password | CP-01 |
| Security | `check-images.sh: containers_run_as_non_root` | Every image runs as a non-root user, read-only root filesystem | CP-15 |
| Security | `check-actions-pinned.sh: actions_pinned_to_sha` | Every GitHub Action pinned to a 40-character commit SHA (already true in CI today) | CP-41 (P2) |
| AI rules | `test_agent_stops_after_8_tool_calls` | 9th tool call is never executed, across router, Haiku and Sonnet steps | CP-43 |
| AI rules | `test_agent_hard_timeout_15s` | Slow tool -> loop ends by 15 s with `timed_out` | CP-43 |
| AI rules | `test_every_llm_call_sets_max_tokens` | Code scan + runtime check on the client wrapper | CP-43 |
| AI rules | `test_cost_guard_stops_before_limit` | Loop stops before projected cost exceeds $0.02 | CP-43 |
| AI rules | `test_opus_not_in_live_path` | API refuses to start if a live model setting names Opus | CP-43 |
| AI rules | `test_untrusted_content_is_wrapped_and_tool_args_validated` | Injection defenses in place | CP-32 |
| AI rules | `test_make_eval_reports_quality_latency_cost` | `make eval` output contains all gated metrics in dollars and seconds | CP-37 |

---

## 8. Risks

**Decision:** The biggest risks are (1) the full-answer path on Sonnet approaching the $0.02 and
1.5-second limits, (2) running only the local login until a real provider is chosen, and (3) the MVP
being API-only, so the director sees results through reports and demos rather than screens. Each has
a mitigation below.

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Local dev login only (Q1); a later real provider causes rework | High | Medium | Swappable auth seam (ADR 0008); every endpoint already tested for 401/403/404; swap is one ticket (CP-44, Phase 2); dev login refuses to start outside local/test |
| R2 | Full-path questions on Sonnet with many tool calls approach **$0.02** | Medium | High | Haiku handles refusals and lookups; cost guard stops before the limit; prompt caching; "budget stops <= 5%" gate; fallback mode as a new ticket if needed |
| R3 | Router step plus tools make **<= 1.5 s** first token hard | Medium | Medium | Router `max_tokens` 64 and 2 s timeout; parallel tool calls; small tool results; measured in every agent PR |
| R4 | Redis crash loses up to about **1 s** of accepted telemetry (`appendfsync everysec`) | Low | Low | Accepted for sampled telemetry; business events use the Postgres outbox (ADR 0001) |
| R5 | Raw telemetry growth: at 2,000 points/second, about **173 million rows/day** | Medium | High | Daily partitions; 14-day raw retention (approved Q6) via CP-42 in Phase 2; the 2,000/s rate is headroom, not the synthetic steady state |
| R6 | pgvector search filtered by tenant can return fewer than 5 results | Medium | Medium | pgvector iterative index scans, hybrid keyword search, recall@5 gate in `make eval` |
| R7 | Synthetic data is cleaner than reality, so forecast accuracy looks better than it would be | High | Medium | Generator adds gaps, bad rows and step changes; approved targets revisited when real data exists |
| R8 | Prompt injection through uploaded policy documents | Medium | High | Untrusted-content wrapping, read-only allow-listed tools, tenant never a tool argument, 20 injection cases at 100% target |
| R9 | Two approvals over-allocate the same room | Medium | High | Row lock on room during check; concurrency test with 20 parallel approvals |
| R10 | Laptop load tests are noisy, giving flaky SLO results | Medium | Low | Fixed Colima size, warm-up, 3 runs with the median reported (CP-40) |
| R11 | Judge (Sonnet) favours answers written by Sonnet | Medium | Medium | Calibration against 30 director-labelled answers; Opus comparison as a second view |
| R12 | Dependencies added early show up in vulnerability scans | Medium | Low | Trivy gate already in CI; Dependabot (CP-41, Phase 2); pinned versions |
| R13 | 32 MVP PRs plus 22 Phase 2 PRs need director approval | High | Medium | Small PRs (under 800 lines), plain-English summaries, waves let several merge per sitting |
| R14 | Forecast and capacity math defined inconsistently across API, web and assistant | Medium | Medium | Single definitions in docs/metrics.md and one capacity service used by all |
| R15 | MVP has no general rate limits or security headers (CP-26 is Phase 2) | Low (local only) | Medium | MVP runs only on the local stack with synthetic data; planner and ingest have their own limits; CP-26 is first in Phase 2 and required before any non-local deployment |
| R16 | MVP is API-only; the director has no screens | High | Low | Demo through `make dev-token` + scripted requests, eval and backtest reports; director decided (Q12) to keep the MVP API-only and build the planner chat screen first in Phase 2 |

---

## 9. Tickets: MVP and Phase 2

**Decision:** There are **54 tickets** (CP-01 to CP-54). **32 tickets in 8 waves** make up the
**MVP**, and **22 tickets in 4 waves** make up **Phase 2**. Tickets in the same wave can be built at
the same time because **no two of them touch the same file** (each wave's file list proves it). A
ticket only depends on tickets in earlier waves, and every MVP ticket depends only on other MVP
tickets. Ticket IDs are unchanged from the first draft; two Phase 2 tickets are new (CP-53, CP-54,
split out so the MVP does not need the web app). CP-44 (real login) waits for the provider choice at the start of Phase 2 (Q13) and
CP-49 (staging, on the director's home K3s cluster, Q14) comes after it. The director accepted the 32-ticket MVP as laid out.

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
- In ticket details (9.3), "Files: as in overlap check" means the file list given for that ticket in
  its wave in 9.1 or 9.2.

### 9.1 MVP

**What you get at MVP, in plain English.** A working system on the local laptop stack (`make up`),
filled with synthetic data for 3 customers, 12 sites and about 2,500 racks. Telemetry senders can push
power and space readings at up to 2,000 per second; when the system is busy it tells them to wait
instead of breaking, and nothing is lost or stored twice. Planners can request capacity, approvers
can approve or reject it, and the system refuses any approval that would over-commit a room, keeping
a full history of who did what. Every night at 02:00 UTC each site gets a 90-day forecast of peak
power and rack units with best-guess and range, checked against the approved accuracy targets. And
planners can ask the AI assistant questions that mix policy and live numbers ("Can DAL-1 Room B take
120 kW more in March under our loading policy?") and get a cited answer within a few seconds for at
most 2 cents, with `make eval` proving the quality, speed and cost targets. Login is the local test
login. There are no screens yet: you see the MVP through a guided demo, the eval report and the
forecast backtest report. Editing sites and racks, screens, load tests, CI hardening and staging come
in Phase 2.

**Why 32 tickets and not about 20.** The four capabilities themselves are only 9 tickets: ingest
(CP-19, CP-20), reservations (CP-23), nightly forecast (CP-10, CP-24, CP-28) and the planner
API, loop and eval (CP-43, CP-47, CP-37). Each one rests on prerequisites that CLAUDE.md makes
mandatory and that I could not honestly drop:
- **Platform and safety (14):** local stack, workspace and Makefile, threat model, metric definitions,
  schema with tenant isolation, API skeleton, pagination, queue library, retry/timeout library, login
  seam, second schema migration, worker with outbox, idempotency keys, capacity service.
- **Data to run on (2):** inventory seed (there is no site/rack editing in the MVP) and 730 days of
  telemetry history plus policy documents (a forecast needs at least 56 days of history; we cannot wait
  56 days of live ingest).
- **Container images (1):** CP-15, so `make up` runs the API and worker in containers as CLAUDE.md
  requires.
- **Planner prerequisites (6):** document upload and embedding (CP-27), loading the policy documents
  (CP-31), retrieval (CP-30), AI security guards (CP-32), golden questions (CP-33) and the agent's
  tools (CP-36).

**Dependencies pulled in (prerequisites that are not one of the four capabilities):** CP-01, CP-02,
CP-03, CP-04, CP-05, CP-06, CP-07, CP-08, CP-09, CP-12, CP-13, CP-14, CP-15, CP-16, CP-17, CP-18,
CP-25, CP-27, CP-30, CP-31, CP-32, CP-33, CP-36. Dependency corrections made while cutting:
- CP-23 depends on CP-07 (the reservation list is paginated); this was missing in the first draft.
- CP-36 now depends on the services it calls (CP-18, CP-20, CP-23, CP-24), not on the Phase 2
  capacity endpoints (CP-29).
- CP-37 depends on CP-31 (documents must be loaded before `make eval`).
- CP-15 no longer builds the web image, so the MVP does not need the web shell (CP-11). The web image
  moves to new ticket CP-53 (Phase 2).
- The site lookup the agent tools need (site code -> id) lives in the capacity service (CP-18).

**MVP order (build order by wave):**

| MVP wave | Tickets | What it delivers |
|---|---|---|
| M1 | CP-01, CP-02, CP-03, CP-04 | Local stack, workspace, threat model, metric definitions |
| M2 | CP-05, CP-06, CP-07, CP-08, CP-09, CP-10 | Schema and tenant isolation, API skeleton, pagination, queue and retry libraries, forecast model |
| M3 | CP-12, CP-13, CP-14, CP-15, CP-16, CP-17, CP-18 | Login seam, telemetry/forecast/document tables, seed inventory, containers, worker and outbox, idempotency, capacity math |
| M4 | CP-19, CP-20, CP-23, CP-24, CP-25, CP-27 | Telemetry ingest end to end, reservations, forecast read API, history and policy seed, document pipeline |
| M5 | CP-28, CP-30, CP-31, CP-32, CP-33 | Nightly forecast, retrieval, documents loaded, AI guards, golden questions |
| M6 | CP-36, CP-37 | Agent tools, `make eval` |
| M7 | CP-43 | Planner agent loop (router + two models) |
| M8 | CP-47 | Planner question API |

Overlap check per MVP wave (files each ticket touches; no file appears twice in a wave):

- **M1:** CP-01 = `infra/compose/compose.yaml`, `infra/postgres/init/**`, `infra/redis/redis.conf`,
  `infra/scripts/check-stack.sh`, `.env.example`; CP-02 = root `pyproject.toml`, `uv.lock`,
  `.python-version`, `Makefile`, package skeletons (`<pkg>/pyproject.toml`, `<pkg>/src/cp_<pkg>/__init__.py`,
  `<pkg>/tests/conftest.py`, `<pkg>/tests/test_smoke.py`), `worker/src/cp_worker/__main__.py`,
  `core/src/cp_core/settings.py`, `core/src/cp_core/testing/{__init__,containers}.py`;
  CP-03 = `docs/threat-model.md`; CP-04 = `docs/metrics.md`.
- **M2:** CP-05 = `core/alembic.ini`, `core/migrations/{env.py,script.py.mako}`,
  `core/migrations/versions/0001_core_schema.py`, `core/src/cp_core/db/**`, `core/src/cp_core/testing/db.py`,
  `core/tests/db/**`; CP-06 = `api/src/cp_api/{main,app_factory,discovery,problems}.py`,
  `api/src/cp_api/routers/{__init__,health}.py`, `api/src/cp_api/plugins/__init__.py`,
  `api/src/cp_api/schemas/__init__.py`, `api/src/cp_api/contract/{__init__,bundle}.py`,
  `core/src/cp_core/observability/**`, `contracts/openapi/base.yaml`, `contracts/openapi/paths/health.yaml`,
  `contracts/openapi/components/problems.yaml`, `api/tests/core/**`, `core/tests/observability/**`;
  CP-07 = `core/src/cp_core/pagination.py`, `core/tests/test_pagination.py`,
  `contracts/openapi/components/pagination.yaml`; CP-08 = `core/src/cp_core/streams/**`,
  `core/tests/streams/**`; CP-09 = `core/src/cp_core/resilience/**`, `core/tests/resilience/**`;
  CP-10 = `forecast/src/cp_forecast/{series,baseline,model,backtest,metrics}.py`, `forecast/tests/model/**`,
  `reports/backtest/**`.
- **M3:** CP-12 = `api/src/cp_api/auth/{__init__,context,provider,dev_provider,dev_token,permissions}.py`,
  `api/src/cp_api/plugins/auth.py`, `contracts/openapi/components/security.yaml`, `api/tests/auth/**`,
  `api/tests/security/test_route_auth_inventory.py`, `docs/threat-model.md`;
  CP-13 = `core/src/cp_core/db/models/{telemetry,forecasting,knowledge,planner}.py`,
  `core/src/cp_core/telemetry/{__init__,messages}.py`, `core/migrations/versions/0002_telemetry_forecast_knowledge.py`,
  `core/tests/db/test_0002_schema.py`; CP-14 = `seed/src/cp_seed/{generate,inventory,series,load_inventory,cli}.py`,
  `seed/tests/inventory/**`; CP-15 = `infra/docker/{api,worker}.Dockerfile`, `.dockerignore`,
  `infra/compose/compose.yaml`, `infra/scripts/check-images.sh`;
  CP-16 = `worker/src/cp_worker/{__main__,runner,discovery,outbox_relay,inbox}.py`,
  `worker/src/cp_worker/{consumers,schedules,jobs}/__init__.py`, `worker/src/cp_worker/schedules/scheduler.py`,
  `core/src/cp_core/outbox.py`, `worker/tests/runtime/**`; CP-17 = `api/src/cp_api/idempotency.py`,
  `contracts/openapi/components/idempotency.yaml`, `api/tests/idempotency/**`;
  CP-18 = `core/src/cp_core/services/{__init__,capacity}.py`, `core/tests/services/test_capacity.py`,
  `docs/metrics.md`. (CP-05's `core/src/cp_core/db/**` in M2 and CP-13's new model files in M3 are in
  different waves.)
- **M4:** CP-19 = `api/src/cp_api/routers/telemetry_ingest.py`, `api/src/cp_api/schemas/telemetry.py`,
  `core/src/cp_core/telemetry/rack_cache.py`, `contracts/openapi/paths/telemetry-ingest.yaml`,
  `contracts/openapi/components/telemetry.yaml`, `api/tests/telemetry/**`;
  CP-20 = `worker/src/cp_worker/consumers/telemetry.py`, `worker/src/cp_worker/schedules/rollups.py`,
  `core/src/cp_core/telemetry/{writer,rollups}.py`, `core/src/cp_core/services/utilization.py`,
  `core/tests/telemetry/**`, `worker/tests/telemetry/**`;
  CP-23 = `api/src/cp_api/routers/reservations.py`, `api/src/cp_api/schemas/reservations.py`,
  `core/src/cp_core/services/reservations.py`, `core/src/cp_core/domain/{__init__,reservation_state}.py`,
  `worker/src/cp_worker/schedules/reservation_expiry.py`, `worker/src/cp_worker/consumers/events_notifier.py`,
  `contracts/openapi/{paths,components}/reservations.yaml`, `api/tests/reservations/**`,
  `core/tests/services/test_reservations.py`;
  CP-24 = `core/src/cp_core/services/forecasts.py`, `api/src/cp_api/routers/forecasts.py`,
  `api/src/cp_api/schemas/forecasts.py`, `contracts/openapi/{paths,components}/forecasts.yaml`,
  `api/tests/forecasts/**`; CP-25 = `seed/src/cp_seed/load_telemetry.py`, `seed/policies/*.md`,
  `seed/tests/telemetry/**`; CP-27 = `api/src/cp_api/routers/documents.py`,
  `api/src/cp_api/schemas/documents.py`, `contracts/openapi/{paths,components}/documents.yaml`,
  `worker/src/cp_worker/consumers/documents.py`, `core/src/cp_core/ai/{__init__,embeddings,chunking}.py`,
  `api/tests/documents/**`, `worker/tests/documents/**`, `core/tests/ai/**`.
- **M5:** CP-28 = `forecast/src/cp_forecast/job.py`, `worker/src/cp_worker/schedules/nightly_forecast.py`,
  `api/src/cp_api/routers/forecast_runs.py`, `api/src/cp_api/schemas/forecast_runs.py`,
  `contracts/openapi/{paths,components}/forecast-runs.yaml`, `forecast/tests/job/**`, `api/tests/forecast_runs/**`;
  CP-30 = `agent/src/cp_agent/retrieval.py`, `agent/tests/retrieval/**`;
  CP-31 = `seed/src/cp_seed/load_documents.py`, `seed/tests/documents/**`;
  CP-32 = `agent/src/cp_agent/guards.py`, `agent/tests/guards/**`, `docs/threat-model.md`;
  CP-33 = `evals/golden/**`.
- **M6:** CP-36 = `agent/src/cp_agent/tools/**`, `agent/tests/tools/**`;
  CP-37 = `evals/src/cp_evals/**`, `evals/config/**`, `evals/tests/**`.
- **M7:** CP-43 = `agent/src/cp_agent/{loop,router,llm,registry,answer}.py`,
  `agent/src/cp_agent/prompts/{router,answer}.md`, `agent/tests/loop/**`.
- **M8:** CP-47 = `api/src/cp_api/routers/planner.py`, `api/src/cp_api/schemas/planner.py`,
  `core/src/cp_core/services/planner_history.py`, `contracts/openapi/{paths,components}/planner.yaml`,
  `api/tests/planner/**`.

MVP is done when: all 32 tickets are merged, `make up && make seed` works from scratch, `make eval`
passes every gated target in section 6, the CP-10/CP-28 backtest meets section 6, and every named
test in section 7 marked with an MVP ticket passes.

### 9.2 Phase 2

**Decision:** Phase 2 adds the web screens, editing of sites/rooms/racks, the general security
plugins, load and failure tests, CI hardening, data retention, the nightly eval in CI, the real login
provider (when chosen) and staging (deferred). It starts after the MVP is accepted.

**Phase 2 order (director decision, Q12 and Q13):**

1. **Decision point:** at the start of Phase 2 the director chooses the real login provider (unblocks CP-44).
2. **CP-26 first** (rate limits and security headers; must land before anything leaves the laptop).
3. **Planner chat screen next:** CP-50 is the first screen built, immediately after CP-26. Its prerequisites
   are pulled forward in this order: CP-11, CP-21, CP-22 (wave P1), CP-35 and CP-53 (wave P2), then CP-50.
   Other Phase 2 tickets fill the same waves in parallel but do not delay this chain.
4. Staging (CP-49) deploys to the director's home K3s cluster (Raspberry Pi, arm64) once CP-44 is done.

The wave grouping below is unchanged, so the no-shared-files check still holds.

| Phase 2 wave | Tickets | What it delivers |
|---|---|---|
| P1 | CP-11, CP-21, CP-22, CP-26, CP-41, CP-42, CP-45 | Web shell, sites/rooms and racks editing, rate limits and security headers, CI hardening, retention and backfill, failure tests |
| P2 | CP-29, CP-34, CP-35, CP-44 (waits for provider choice), CP-48, CP-53 | Capacity/utilization endpoints, cross-endpoint integration tests, typed web client + Sites screens, real login, nightly eval in CI, web container |
| P3 | CP-38, CP-39, CP-40, CP-46, CP-49 (deferred), CP-50 | Reservation, forecast, documents and planner screens, load tests, staging |
| P4 | CP-51, CP-52, CP-54 | Final acceptance report, security audit, Sites screen capacity panel |

Overlap check per Phase 2 wave:

- **P1:** CP-11 = `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `turbo.json`, `apps/web/**`;
  CP-21 = `api/src/cp_api/routers/{sites,rooms}.py`, `api/src/cp_api/schemas/{sites,rooms}.py`,
  `core/src/cp_core/services/{sites,rooms}.py`, `contracts/openapi/{paths,components}/{sites,rooms}.yaml`,
  `api/tests/{sites,rooms}/**`; CP-22 = `api/src/cp_api/routers/racks.py`, `api/src/cp_api/schemas/racks.py`,
  `core/src/cp_core/services/racks.py`, `contracts/openapi/{paths,components}/racks.yaml`, `api/tests/racks/**`;
  CP-26 = `api/src/cp_api/plugins/{rate_limit,security_headers,cors,body_limit}.py`,
  `api/src/cp_api/security/**`, `api/tests/security/test_{rate_limit,headers,cors,body_limit}.py`,
  `docs/threat-model.md`; CP-41 = `.github/workflows/ci.yml`, `.github/dependabot.yml`,
  `infra/scripts/check-actions-pinned.sh`; CP-42 = `worker/src/cp_worker/schedules/{partition_maintenance,retention}.py`,
  `worker/src/cp_worker/jobs/backfill_rollups.py`, `core/src/cp_core/jobs/{__init__,checkpoints}.py`,
  `worker/tests/maintenance/**`; CP-45 = `tests/failure/**`, `reports/failure/**`.
- **P2:** CP-29 = `api/src/cp_api/routers/{capacity,utilization}.py`, `api/src/cp_api/schemas/capacity.py`,
  `contracts/openapi/paths/{capacity,utilization}.yaml`, `contracts/openapi/components/capacity.yaml`,
  `api/tests/capacity/**`; CP-34 = `tests/integration/**`, `reports/standards/**`;
  CP-35 = `apps/web/lib/api/**`, `apps/web/app/sites/**`, `apps/web/e2e/sites.spec.ts`;
  CP-44 = `api/src/cp_api/auth/providers/**`, `api/tests/auth/providers/**`, `docs/threat-model.md`,
  `core/migrations/versions/0003_users_memberships.py`, `core/src/cp_core/db/models/identity.py`,
  `core/tests/db/test_0003_users.py` (the only P2 ticket with a migration, ADR 0004 rule 7)
  (and `uv.lock` if a library is needed; it is the only P2 ticket allowed to touch it);
  CP-48 = `.github/workflows/nightly-eval.yml`; CP-53 = `infra/docker/web.Dockerfile`,
  `infra/compose/compose.yaml`.
- **P3:** CP-38 = `apps/web/app/reservations/**`, `apps/web/e2e/reservations.spec.ts`;
  CP-39 = `apps/web/app/forecasts/**`, `apps/web/e2e/forecasts.spec.ts`; CP-40 = `load/**`, `reports/load/**`;
  CP-46 = `apps/web/app/documents/**`, `apps/web/e2e/documents.spec.ts`;
  CP-49 = `.github/workflows/deploy-staging.yml`, `infra/staging/**`;
  CP-50 = `apps/web/app/planner/**`, `apps/web/e2e/planner.spec.ts`.
- **P4:** CP-51 = `tests/acceptance/**`, `reports/acceptance/**`; CP-52 = no repository files (opens
  issues only); CP-54 = `apps/web/app/sites/[siteId]/capacity/**`, `apps/web/e2e/site-capacity.spec.ts`
  (CP-35 created `apps/web/app/sites/**` in P2, a different wave).

### 9.3 Ticket details (in ID order)

Each ticket shows whether it is **MVP (wave Mn)** or **Phase 2 (wave Pn)**.

#### CP-01: Local data stack (Postgres + pgvector, Redis), database roles, environment template
- MVP (M1) | Owner: release | Risk: risk:high (infra) | Size: ~250 lines | Depends on: none
- Files: as in overlap check
- Acceptance:
  1. `docker compose -f infra/compose/compose.yaml up -d` starts Postgres 16 with pgvector on host port **15432** and Redis 7 on host port **16379**; both healthchecks pass within **60 s**.
  2. Roles `cp_owner`, `cp_app` (NOBYPASSRLS, no CREATE) and `cp_scheduler` exist; `vector` extension is installed; database `cp_eval` also exists.
  3. Redis runs with `appendonly yes`, `appendfsync everysec`, `maxmemory-policy noeviction` (verified by `CONFIG GET` in `check-stack.sh`).
  4. `.env.example` lists every planned variable name with empty or dummy values only: DB URLs per role, `REDIS_URL`, `CP_ENV`, `CP_AUTH_PROVIDER`, `CP_AUTH_DEV_SIGNING_KEY`, `CURSOR_SIGNING_KEY`, `ANTHROPIC_API_KEY`, `CP_AGENT_ROUTER_MODEL` (`claude-haiku-5-5`), `CP_AGENT_ANSWER_MODEL` (`claude-sonnet-5-5`), `CP_EVAL_COMPARE_MODEL` (`claude-opus-5-5`), `CP_EVAL_JUDGE_MODEL` (`claude-sonnet-5-5`), `CP_EMBED_PROVIDER` (`voyage`), `CP_EMBED_MODEL`, `CP_EMBED_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT`, stream caps, worker concurrency.
  5. Both images are multi-arch (linux/arm64 and linux/amd64), pinned by digest; containers run as non-root users.
  6. No host ports other than 15432, 16379, 18000 and 18080 are used by the stack.
  7. Postgres and Redis publish their ports on **127.0.0.1 only** (for example `127.0.0.1:15432:5432`), so nothing else on the network can reach them. This matters because workers trust the `tenant_id` inside queue messages (ADR 0008), so open Redis access would bypass tenant isolation.
  8. Redis requires a password: an ACL user `cp_app` with password from `REDIS_PASSWORD`, and the `default` user disabled. `.env.example` adds `REDIS_PASSWORD` (empty) and `REDIS_URL` includes the user name. Database role passwords also come only from `.env`.
  9. `check-stack.sh` proves both, as named checks: `check-stack.sh: db_and_redis_bound_to_localhost` (inspects published port bindings; fails if any is `0.0.0.0` or `::`) and `check-stack.sh: redis_requires_auth` (`PING` without credentials must return `NOAUTH`; with credentials must return `PONG`).

#### CP-02: Python workspace, dependency baseline, Makefile, lint and type-check config
- MVP (M1) | Owner: builder | Risk: risk:high (dependencies) | Size: ~550 lines | Depends on: none
- Files: as in overlap check (packages: `core`, `api`, `worker`, `forecast`, `agent`, `seed`, `evals`, `load`, `tests`)
- Acceptance:
  1. `uv sync` installs the workspace on arm64 and amd64 with Python 3.12; `uv.lock` committed.
  2. Baseline dependencies declared with a one-line reason each in the PR: FastAPI, Uvicorn, Pydantic v2, pydantic-settings, SQLAlchemy 2, Alembic, psycopg 3, redis-py, httpx, OpenTelemetry (API, SDK, FastAPI, SQLAlchemy, httpx, Redis instrumentations), statsmodels, numpy, pandas, anthropic, pgvector (Python), python-multipart, PyJWT; dev: pytest, pytest-asyncio, Testcontainers, ruff, mypy, Locust, hypothesis. (Voyage AI is called through httpx; no extra SDK.)
  3. `make check` runs ruff and mypy (strict on `core`, `api`, `agent`) and, when `package.json` exists, `pnpm turbo lint typecheck`; it passes on the skeleton.
  4. `make test` runs pytest across all workspace members and, when present, `pnpm turbo test`; it passes.
  5. Makefile has targets `up`, `down`, `check`, `test`, `eval`, `migrate`, `seed`, `dev-token`, `load`, `backtest`, `check-images`; `make up` runs compose from `infra/compose/compose.yaml` then `make migrate`; `make eval` prints "no evals yet" and exits 0 until `cp_evals` has a runner.
  6. Shared fixtures start throwaway Postgres (pgvector) and Redis containers for tests in under **30 s**.

#### CP-03: Threat model v1
- MVP (M1) | Owner: security | Risk: risk:high (security agent rule) | Size: ~400 lines | Depends on: none
- Files: `docs/threat-model.md`
- Acceptance:
  1. Lists assets, actors (5 roles plus anonymous and compromised-sender), entry points (every endpoint in spec section 3, queues, worker, LLM and embedding calls, uploads), and trust boundaries.
  2. For each entry point, STRIDE threats (spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege) with mitigation and the named test that proves it.
  3. Marks `/healthz` and `/readyz` as the only public endpoints.
  4. Has a section on AI threats: prompt injection via documents and questions, tool misuse, data exfiltration, cost abuse, and an Opus model accidentally configured in the live path.
  5. Records "local dev login only (Q1)" with the ADR 0008 compensating controls, and that the MVP is local-only until CP-26 and CP-44 land.

#### CP-04: Metric definitions
- MVP (M1) | Owner: analytics | Risk: risk:low (docs) | Size: ~250 lines | Depends on: none
- Files: `docs/metrics.md`
- Acceptance:
  1. Defines capacity, committed, used, headroom (kW and RU, room and site level), utilization %, peak kW, WAPE, MAPE, interval coverage, recall@5, faithfulness, correctness, routing accuracy, cost per question; each with name, plain-English meaning, formula, grain, owner.
  2. States that forecasts model organic demand and committed reservations are layered on top (ADR 0006).
  3. Each metric names the code module that will compute it (to be filled by later tickets).

#### CP-05: Core schema, tenant isolation and migration 0001
- MVP (M2) | Owner: data-engineer | Risk: risk:high (migrations, tenancy) | Size: ~750 lines | Depends on: CP-01, CP-02
- Files: as in overlap check (`core/src/cp_core/db/models/__init__.py` auto-imports model modules)
- Acceptance:
  1. Migration 0001 creates `tenants`, `sites`, `rooms`, `racks`, `reservations`, `allocations`, `reservation_events`, `outbox_events`, `processed_events`, `idempotency_keys`, `audit_log`, `job_checkpoints` exactly as in spec section 2, with composite tenant foreign keys and indexes leading on `tenant_id`.
  2. RLS is enabled and forced on every tenant-owned table with `USING` and `WITH CHECK` on `app.tenant_id`.
  3. `tenant_session(tenant_id)` sets `SET LOCAL app.tenant_id` per transaction; using a session without it raises.
  4. Named tests pass: `test_every_tenant_table_has_tenant_id_and_forced_rls`, `test_rls_blocks_query_without_tenant_setting`, `test_app_role_cannot_bypass_rls`, `test_child_cannot_reference_other_tenant_parent`, `test_all_migrations_upgrade_downgrade_roundtrip`, `test_single_alembic_head`, `test_no_destructive_ops_outside_contract_migrations`.
  5. Migration sets `lock_timeout = 5s` and `statement_timeout = 60s`; PR shows upgrade and downgrade output and `EXPLAIN` for the list-query indexes.

#### CP-06: API skeleton, auto-discovery, error format, JSON logs and tracing, OpenAPI scaffolding
- MVP (M2) | Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-02
- Files: as in overlap check
- Acceptance:
  1. Every module in `cp_api/routers/` and `cp_api/plugins/` is loaded automatically; `test_route_inventory_matches_contract` fails if a route is missing from the bundled contract or vice versa.
  2. `bundle.py` merges `base.yaml` + `paths/*.yaml` + `components/*.yaml` into `contracts/openapi/dist/openapi.json` (git-ignored) and validates it as OpenAPI 3.1.
  3. All errors return RFC 9457 problem+json with `trace_id`; unhandled exceptions return 500 with no stack trace (tested).
  4. `test_logs_are_json_with_trace_id` passes; incoming `traceparent` is continued and `X-Trace-Id` returned.
  5. `GET /healthz` returns 200 in under **50 ms**; `GET /readyz` returns 503 when Postgres or Redis is unreachable.

#### CP-07: Keyset pagination helper (ADR 0002)
- MVP (M2) | Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-02
- Files: as in overlap check
- Acceptance:
  1. Helper builds `WHERE (sort_key, id) > (...) ORDER BY sort_key, id LIMIT limit+1` for ascending and descending sorts and returns `{items, next_cursor}`.
  2. Cursors are base64url JSON with HMAC-SHA256 over sort, direction, last values and filter hash.
  3. Named tests pass: `test_page_size_capped_at_100`, `test_keyset_pagination_no_duplicates_or_gaps_under_concurrent_inserts` (10,000 rows, inserts during paging), `test_ties_broken_by_id`, `test_tampered_cursor_rejected`.
  4. Default limit is **50**; limit outside 1..100 raises a validation error mapped to 422.

#### CP-08: Redis Streams library: bounded producer, consumer groups, dead-letter, reclaim
- MVP (M2) | Owner: builder | Risk: risk:high (queues) | Size: ~650 lines | Depends on: CP-01, CP-02
- Files: as in overlap check
- Acceptance:
  1. `enqueue_batch(stream, entries)` appends all entries or none, atomically refusing if `XLEN + n > cap`; returns current depth and a suggested `Retry-After` of **1..30 s**. `test_enqueue_is_atomic_and_never_exceeds_cap` passes with 100 concurrent producers.
  2. Consumer base reads with a consumer group, at most **500** entries per read, acknowledges and deletes on success, and reclaims entries idle for more than **60 s**.
  3. Failures retry with backoff (0.5 s doubling, max 30 s, jitter); after **5** deliveries the entry moves to `cp:dlq:<stream>` with error, attempts and `traceparent`; `test_poison_message_moves_to_dead_letter_after_5_deliveries` passes.
  4. DLQ capped at **100,000** entries; `python -m cp_core.streams.cli replay <stream> [--limit N]` re-queues DLQ entries respecting the cap.
  5. Each entry carries `tenant_id` and `traceparent`; consumer spans link to the producer span.

#### CP-09: Resilience library: retries, timeouts, outbound HTTP client
- MVP (M2) | Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-02
- Files: as in overlap check
- Acceptance:
  1. `retry(...)` implements exponential backoff with full jitter, max attempts, max delay, and retries only on retryable errors (timeouts, 429, 5xx); honours `Retry-After`.
  2. `make_http_client(...)` requires connect and total timeouts (defaults **2 s** connect, **10 s** total) and adds an `Idempotency-Key` to retried non-GET requests.
  3. Named tests pass: `test_retry_uses_exponential_backoff_with_jitter`, `test_retry_stops_after_max_attempts`, `test_every_outbound_call_has_timeout` (fails if `httpx.Client(`, `httpx.AsyncClient(` or `anthropic.Anthropic(` appears outside this module).

#### CP-10: Forecast model, seasonal naive baseline and backtest harness (ADR 0006)
- MVP (M2) | Owner: analytics | Risk: risk:medium | Size: ~700 lines | Depends on: CP-02
- Files: as in overlap check
- Acceptance:
  1. `fit_forecast(series, horizon=90)` returns 90 daily points with `yhat`, `lo80`, `hi80`, `lo95`, `hi95`; uses ETS damped + weekly seasonality, MSTL + ETS at >= **730** days, refuses under **56** days.
  2. Seasonal naive baseline and rolling-origin backtest (every 14 days over 180 days, >= 6 origins) report WAPE, MAPE and coverage by horizon bucket.
  3. On the synthetic test series (fixed seed) the model meets the approved targets in spec section 6, including beating the baseline WAPE by >= **15%**; if not, the PR states the gap in plain English and stops for director review.
  4. One site-metric fit completes in under **1 s** on the reference laptop.
  5. `make backtest` writes a target-vs-actual table to `reports/backtest/`.

#### CP-11: Web shell (pnpm + Turborepo + Next.js)
- Phase 2 (P1) | Owner: builder | Risk: risk:high (dependencies) | Size: ~600 lines | Depends on: CP-02
- Files: as in overlap check (layout, navigation with Sites, Reservations, Forecasts, Documents, Planner placeholder pages, API base config, Vitest config, Playwright config, `apps/web/e2e/smoke.spec.ts`)
- Acceptance:
  1. `pnpm install` and `pnpm turbo build lint typecheck test` pass; lock file committed.
  2. Dependencies front-loaded with reasons: Next.js, React, TypeScript, Vitest, Playwright, openapi-typescript, openapi-fetch, a chart library (Recharts), Zod.
  3. Dev server runs on port **18080**; navigation shows all 5 sections with placeholder pages.
  4. `apps/web/package.json` has a `gen:api` script that generates types from `contracts/openapi/dist/openapi.json`.
  5. Playwright smoke test loads every section without console errors.

#### CP-12: Swappable auth seam, deny-by-default, permission matrix, authorization test harness (ADR 0008)
- MVP (M3) | Owner: security | Risk: risk:high (auth) | Size: ~650 lines | Depends on: CP-03, CP-06
- Files: as in overlap check
- Acceptance:
  1. Every request passes through `AuthProvider.authenticate` and yields `AuthContext(tenant_id, user_id, roles)`; missing or invalid token -> 401; `require_role(...)` -> 403.
  2. `dev` provider (the MVP login, Q1) validates HS256 tokens with **15-minute** expiry; `make dev-token TENANT=<slug> ROLES=<list>` prints one; `test_dev_auth_refuses_non_local_env` passes.
  3. `test_every_route_requires_auth_unless_public` passes (only `/healthz`, `/readyz` public).
  4. `test_every_route_has_authorization_test` collects `AUTHZ_COVERED` declarations from all test modules and fails for any uncovered route.
  5. Permission matrix in spec section 2 implemented as data in `permissions.py`, with a test per row, including "nobody approves their own request".
  6. Threat model updated; tokens are never logged (test asserts redaction).

#### CP-13: Migration 0002: telemetry, rollups, forecasts, documents, planner history
- MVP (M3) | Owner: data-engineer | Risk: risk:high (migrations, tenancy) | Size: ~700 lines | Depends on: CP-05
- Files: as in overlap check
- Acceptance:
  1. Creates `telemetry_points` (partitioned by day), `telemetry_hourly`, `telemetry_daily`, `forecast_runs`, `forecasts`, `forecast_points`, `policy_documents`, `document_chunks` (`vector(1024)` for Voyage AI, HNSW cosine, GIN on tsv), `planner_questions` (with `route` and `models_used`), all tenant-owned with forced RLS.
  2. Creates partitions from 7 days back to **30 days** ahead (so the MVP runs without CP-42) and two SQL functions: `cp_create_telemetry_partitions(days_ahead int)` (used by CP-20) and `cp_drop_telemetry_partitions_before(cutoff date)` (used by CP-42's retention job; refuses any cutoff newer than today minus **14 days**, the approved raw retention, Q6). Both are `SECURITY DEFINER` (they run with their owner's rights), owned by `cp_owner`, with a fixed `search_path = pg_catalog, public, pg_temp`, and every table they touch is written with its schema (e.g. `public.telemetry_points`) so nothing can be swapped in by name. `EXECUTE` on the create function is granted to `cp_app`; `EXECUTE` on the drop function is granted **only** to `cp_scheduler`, never to `cp_app` (and revoked from `PUBLIC`). `cp_app` is **never** granted `CREATE`, DDL rights or privileges on individual partitions. A test (`test_app_role_cannot_create_or_drop_partitions_directly`) proves `cp_app` cannot run `CREATE TABLE ... PARTITION OF` or `DROP TABLE` itself but can call the create function; `test_retention_function_refuses_cutoff_inside_14_days` proves the drop function refuses a cutoff inside 14 days and that `cp_app` cannot call it while `cp_scheduler` can.
  3. `messages.py` defines the versioned telemetry stream message (`v`, `tenant_id`, `rack_id`, `room_id`, `site_id`, `metric`, `ts`, `value`, `batch_id`, `traceparent`) with Pydantic validation.
  4. Existing tenancy and migration named tests now cover the new tables and pass; upgrade/downgrade round trip passes on seeded data.
  5. PR shows `EXPLAIN` proving index use for `(tenant_id, rack_id, metric, ts)` and `(tenant_id, room_id, metric, day)` lookups.

#### CP-14: Synthetic data generator and inventory/reservation loader
- MVP (M3) | Owner: data-engineer | Risk: risk:medium | Size: ~650 lines | Depends on: CP-05
- Files: as in overlap check
- Acceptance:
  1. `make seed SEED=42` loads 3 tenants, 12 sites, 4-8 rooms per site, 20-60 racks per room, and about 150 reservations across all states; identical results on every run with the same seed.
  2. `series.py` produces deterministic daily and per-minute kW and RU series with trend, weekly and yearly seasonality, step changes, 1% gaps and 0.5% bad rows (pure functions, no DB).
  3. Loader is idempotent: running it twice produces no duplicates (row counts equal).
  4. Data quality tests: row counts per table, no nulls in required fields, referential integrity, every reservation's committed kW within room capacity.
  5. The CLI auto-discovers `load_*.py` modules so later loaders need no CLI edits.

#### CP-15: API and worker container images and compose app services
- MVP (M3) | Owner: release | Risk: risk:high (infra) | Size: ~300 lines | Depends on: CP-01, CP-06
- Files: as in overlap check (the web image moved to CP-53)
- Acceptance:
  1. API and worker images build for linux/arm64 and linux/amd64 with `docker buildx`; base images pinned by digest; slim or distroless runtime.
  2. Containers run as a non-root user with read-only root filesystem and all Linux capabilities dropped; `make check-images` verifies this and fails otherwise.
  3. `make up` starts the API on host port **18000** and the worker with no host port; all healthy within **90 s** on the reference laptop. Port **18080** stays reserved for the web app (CP-53).
  4. No secrets in image layers; Trivy finds zero unfixed HIGH or CRITICAL issues.

#### CP-16: Worker runtime, transactional outbox and outbox relay
- MVP (M3) | Owner: builder | Risk: risk:high (queues) | Size: ~650 lines | Depends on: CP-05, CP-08
- Files: as in overlap check
- Acceptance:
  1. Worker auto-discovers consumers in `consumers/` and schedules in `schedules/`; concurrency per consumer is configurable (default **4**); `test_worker_concurrency_never_exceeds_configured_limit` passes.
  2. `outbox.add(session, stream, event_type, payload)` writes to `outbox_events` in the caller's transaction; relay publishes in batches of **100** with `FOR UPDATE SKIP LOCKED` and marks published.
  3. `test_outbox_event_published_only_after_commit` and `test_outbox_relay_crash_does_not_lose_or_duplicate_effects` pass.
  4. `inbox.once(consumer, event_id)` records processed events so repeats are skipped.
  5. Scheduler uses a Redis lock so only one worker runs each scheduled job; graceful shutdown finishes in-flight items within **10 s**.

#### CP-17: Idempotency keys for API writes
- MVP (M3) | Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-05, CP-06
- Files: as in overlap check
- Acceptance:
  1. A reusable dependency stores `(tenant_id, user_id, key)` with request hash and response for **24 h**; `POST` without the header -> 422.
  2. `test_idempotency_key_replay_returns_original_response` and `test_idempotency_key_reuse_with_different_body_returns_422` pass; concurrent duplicate in flight -> 409. An in-progress key older than the endpoint's hard timeout plus **60 s** (a crashed first attempt) is treated as failed and the retry runs (section 3; proven for the assistant by `test_in_progress_idempotency_key_expires_after_timeout_and_can_retry` in CP-47).
  3. The same key used by two different tenants, or by two users in the same tenant, does not collide (`test_idempotency_key_scoped_per_tenant_and_user`).
  4. Expired keys are purged by a batched delete function (5,000 rows per batch).

#### CP-18: Capacity and headroom service
- MVP (M3) | Owner: analytics | Risk: risk:medium | Size: ~450 lines | Depends on: CP-04, CP-05
- Files: as in overlap check
- Acceptance:
  1. Computes per-room and per-site capacity, committed (allocated + approved-unallocated) and headroom for kW and RU exactly as defined in docs/metrics.md; "used" is an optional input supplied by the utilization service.
  2. `check_fits(room or rack, kw, ru)` returns fit/no-fit with the limiting dimension; used by reservations.
  3. `resolve_site(ctx, site_code)` returns the site id for the current tenant or not-found (used by agent tools).
  4. Tests cover empty rooms, full rooms, archived racks, released allocations and rack-level vs room-level requests.
  5. Room-level capacity query runs in under **20 ms** on seeded data (PR shows `EXPLAIN ANALYZE`); docs/metrics.md updated with the module path for each capacity metric.

#### CP-19: Telemetry ingest endpoint with backpressure and per-item status
- MVP (M4) | Owner: builder | Risk: risk:high (queues) | Size: ~550 lines | Depends on: CP-08, CP-12, CP-13, CP-17
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/telemetry/batches` (role `ingestor`) validates 1-500 points, body <= 256 KB, `ts` window (+5 min / -7 days), value ranges, and rack ownership via a per-tenant rack-id cache refreshed every **60 s**.
  2. Returns **202** with per-item status; `test_batch_ingest_returns_per_item_status` passes.
  3. Full queue -> **429** + `Retry-After` 1..30 s and nothing enqueued (`test_ingest_returns_429_with_retry_after_when_queue_full`); Redis down -> **503** + `Retry-After: 5`.
  4. `Idempotency-Key` replay within 24 h returns the original response without re-enqueueing (stored in Redis under `cp:idem:telemetry:<tenant_id>:<key>`, so the key is always scoped to the sending customer). The same key sent by two different tenants does not collide: both batches are stored and neither gets the other's response (`test_telemetry_idempotency_key_scoped_per_tenant`).
  5. Endpoint p95 <= **400 ms** for 500-point batches in a local micro-benchmark included in the PR.
  6. Authorization tests: non-ingestor -> 403; rack from another tenant -> item `rejected` with reason `unknown_rack`.

#### CP-20: Telemetry consumer, rollups and utilization service
- MVP (M4) | Owner: data-engineer | Risk: risk:high (queues) | Size: ~700 lines | Depends on: CP-13, CP-16
- Files: as in overlap check
- Acceptance:
  1. Consumer writes points with multi-row `INSERT ... ON CONFLICT DO NOTHING` in batches of up to **500**, under the message's tenant; re-delivering a batch 3 times stores each point once.
  2. Invalid or failing messages retry, then dead-letter after 5 deliveries; valid messages in the same read still commit.
  3. Rollup schedule upserts hourly rollups for the last 2 hours every **5 min** and daily rollups at **00:15 UTC**; idempotent on re-run. The daily step also calls `cp_create_telemetry_partitions(14)` so the MVP never runs out of partitions (full maintenance and retention arrive in CP-42). It calls the `SECURITY DEFINER` function owned by `cp_owner` (CP-13); `cp_app` gets no extra DDL rights for this.
  4. `utilization.py` returns latest and windowed used kW/RU per room and site from rollups, paginated by keyset.
  5. `test_trace_spans_api_to_queue_to_worker` passes; freshness p95 <= **10 s** at 2,000 points/second in an integration test of 60 s.

#### CP-21: Sites and rooms CRUD
- Phase 2 (P1) | Owner: builder | Risk: risk:medium | Size: ~700 lines | Depends on: CP-07, CP-12, CP-17
- Files: as in overlap check
- Acceptance:
  1. Endpoints for sites and rooms in spec section 3 implemented with keyset pagination, idempotency on POST, and admin-only writes.
  2. Room `capacity_kw` sum may not exceed site `capacity_kw` (422); archiving a site or room with active reservations -> 409.
  3. Unique site code per tenant and room name per site enforced (409 on duplicate).
  4. Authorization tests per route including cross-tenant -> 404; `AUTHZ_COVERED` declared.
  5. Writes record `audit_log` entries.

#### CP-22: Racks CRUD
- Phase 2 (P1) | Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-07, CP-12, CP-17
- Files: as in overlap check
- Acceptance:
  1. Rack endpoints in spec section 3 implemented with pagination, idempotency, admin-only writes.
  2. `capacity_ru` 1..60, `capacity_kw` > 0; label unique per room; archiving a rack with active allocations -> 409.
  3. Authorization tests per route including cross-tenant -> 404.
  4. Writes record `audit_log` entries.

#### CP-23: Reservation workflow (request -> approve -> allocate)
- MVP (M4) | Owner: builder | Risk: risk:medium | Size: ~780 lines | Depends on: CP-07, CP-12, CP-16, CP-17, CP-18
- Files: as in overlap check
- Acceptance:
  1. State machine allows only the transitions in spec section 2; others -> 409 `invalid-transition`; each transition writes `reservation_events` and an outbox event in the same transaction.
  2. Approve and allocate lock the room row and call `check_fits`; 20 concurrent approvals of 10 kW into a room with 100 kW headroom -> exactly **10** succeed, 10 get 409 `capacity-exceeded`.
  3. Room-level allocation auto-picks racks with enough free kW and RU (fewest racks first) or accepts an explicit rack list; rack-level allocates that rack only.
  4. Nobody, including admins, can approve their own request (approved Q5); requesters can cancel only their own.
  5. Expiry schedule moves `approved` reservations not allocated within **14 days** (approved Q5, configurable) to `expired`, in batches of 500.
  6. `events_notifier` consumer logs a structured "notification" per event using `inbox.once` (stand-in for email later).
  7. List endpoint uses the CP-07 helper; authorization tests per route and role including cross-tenant -> 404.

#### CP-24: Forecast read service and API
- MVP (M4) | Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-12, CP-13
- Files: as in overlap check
- Acceptance:
  1. `GET /v1/sites/{site_id}/forecasts/latest?metric=` returns 90 points with intervals, model name, run id and `generated_at`; 404 if none; `status=insufficient_history` returned as 200 with empty points.
  2. Read p95 <= **200 ms** on seeded data (micro-benchmark in PR).
  3. Service function `latest_forecast(ctx, site_id, metric, horizon_days<=90)` is reusable by the agent tools.
  4. Authorization tests including cross-tenant -> 404.

#### CP-25: Seed telemetry history and synthetic policy documents
- MVP (M4) | Owner: data-engineer | Risk: risk:medium | Size: ~600 lines (documents count as content) | Depends on: CP-13, CP-14
- Files: as in overlap check
- Acceptance:
  1. Loads 730 days of daily and 30 days of hourly rollups for every seeded room, and 2 days of raw 60-second points, deterministic with `SEED=42`, idempotent on re-run.
  2. Writes the 6 approved synthetic policy documents (Q8: power loading policy, reservation and approval policy, rack standards, change-freeze calendar, capacity planning guideline, cooling guideline), each 1,500-4,000 words, clearly marked synthetic, with facts the golden set can reference.
  3. Data quality tests: freshness of latest raw point, no gaps beyond the planned 1%, bad rows flagged not loaded.
  4. Loading completes in under **5 min** on the reference laptop.

#### CP-26: Rate limits, secure headers, CORS, body size limits
- Phase 2 (P1) | Owner: security | Risk: risk:high (security) | Size: ~550 lines | Depends on: CP-06, CP-12
- Files: as in overlap check
- Acceptance:
  1. Rate limit per tenant (**50 requests/second**, burst 100) and per IP for unauthenticated requests (**20 requests/second**) using Redis; exceed -> 429 + `Retry-After`; `test_rate_limit_per_tenant` passes.
  2. `test_security_headers_present`: Content-Security-Policy, Strict-Transport-Security, X-Content-Type-Options, frame-ancestors 'none', Referrer-Policy.
  3. CORS allow-list contains only `http://localhost:18080` by default (configurable); other origins rejected.
  4. Bodies over **1 MB** (2 MB for document upload) -> 413 before parsing.
  5. Threat model updated for all MVP endpoints and the Phase 2 inventory endpoints.

#### CP-27: Document upload, chunking and Voyage AI embedding pipeline
- MVP (M4) | Owner: builder | Risk: risk:high (queues, external AI call) | Size: ~750 lines | Depends on: CP-09, CP-12, CP-13, CP-16, CP-17
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/documents` accepts `.md`/`.txt` up to 2 MB, dedupes by SHA-256, stores the document `pending` and an outbox event to `cp:documents` (cap **500**, 429 when full) in one transaction.
  2. Consumer chunks to about **500 tokens** with **50-token** overlap, embeds in batches through the resilience client (timeouts, retries), writes chunks and tsv, marks `embedded`; failures retry then dead-letter and mark `failed`.
  3. `Embedder` interface with a deterministic `fake` provider (tests) and a Voyage AI adapter (Q3); start-up check fails if the configured model's vector size is not **1,024**.
  4. A 2 MB document is searchable within **120 s** locally; delete removes chunks.
  5. Authorization tests: only admin uploads/deletes; cross-tenant -> 404.

#### CP-28: Nightly forecast job, schedule and manual trigger
- MVP (M5) | Owner: analytics | Risk: risk:medium | Size: ~600 lines | Depends on: CP-10, CP-16, CP-20, CP-24
- Files: as in overlap check
- Acceptance:
  1. Job loops over tenants (as `cp_scheduler`), then sites per tenant (as `cp_app` with tenant set), fits `peak_kw` and `ru_used`, writes `forecasts` and `forecast_points`; one site's failure marks it `failed` and the run `partial`, others continue.
  2. Scheduled at **02:00 UTC**; finishes **500 sites x 2 metrics** in under **30 min** (test with synthetic sites; PR shows timing).
  3. `POST /v1/forecast-runs` (admin) enqueues a run (202, idempotent); `GET /v1/forecast-runs` paginated.
  4. Each run is a new `forecast_runs` row and "latest" points to the newest successful forecast; nothing is overwritten.
  5. Plain-English interpretation of backtest results on seeded data in the PR, against the approved targets.

#### CP-29: Capacity summary and utilization endpoints
- Phase 2 (P2) | Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-18, CP-20, CP-21
- Files: as in overlap check
- Acceptance:
  1. `GET /v1/sites/{site_id}/capacity` returns per-room and site totals of capacity, committed, used and headroom for kW and RU using only `capacity.py` and `utilization.py`.
  2. `GET /v1/sites/{site_id}/utilization` supports `metric`, `grain=hour|day`, `from`, `to` (max 400 days), keyset pagination on `ts`.
  3. Both meet read p95 <= **200 ms** on seeded data (micro-benchmark in PR).
  4. Authorization tests including cross-tenant -> 404.

#### CP-30: Hybrid retrieval over policy chunks
- MVP (M5) | Owner: builder | Risk: risk:medium | Size: ~450 lines | Depends on: CP-27
- Files: as in overlap check
- Acceptance:
  1. `search(ctx, query, k<=8)` runs pgvector cosine top 20 and full-text top 20 under tenant RLS, merges with reciprocal rank fusion, returns top k with document title, chunk id and text.
  2. Enables pgvector iterative index scan so tenant filtering still returns k results when they exist (tested with 3 tenants).
  3. Query embedding call has a timeout of **2 s** and one retry; p95 search latency <= **300 ms** locally (excluding Voyage AI time, which is reported separately).
  4. Unit tests with the fake embedder; recall@5 measured by `make eval`.

#### CP-31: Load synthetic policy documents through the API
- MVP (M5) | Owner: data-engineer | Risk: risk:medium | Size: ~200 lines | Depends on: CP-15, CP-25, CP-27
- Files: as in overlap check
- Acceptance:
  1. `make seed` uploads every file in `seed/policies/` for each tenant via `POST /v1/documents` on `localhost:18000` with idempotency keys, then waits until all are `embedded` (timeout **5 min**).
  2. Re-running uploads nothing new (dedupe by hash).
  3. Test verifies chunk counts are greater than 0 for every document.

#### CP-32: AI guards: untrusted-content wrapping, tool argument validation, output checks
- MVP (M5) | Owner: security | Risk: risk:high (AI agent loop, security) | Size: ~400 lines | Depends on: CP-12
- Files: as in overlap check
- Acceptance:
  1. `wrap_untrusted(source, text)` places retrieved text and tool results inside marked data blocks and strips control sequences that mimic system or tool markers.
  2. `validate_tool_call(name, args)` rejects any tool not on the allow-list and any argument failing its schema (including any attempt to pass `tenant_id`).
  3. `check_output(answer)` flags system-prompt leakage and secret-like strings; flagged answers are replaced by a safe refusal.
  4. `test_untrusted_content_is_wrapped_and_tool_args_validated` passes; threat model updated for the planner endpoint, router and tools.

#### CP-33: Golden evaluation datasets
- MVP (M5) | Owner: qa | Risk: risk:low (tests only) | Size: ~700 lines (data) | Depends on: CP-25
- Files: `evals/golden/{retrieval,planner_qa,out_of_scope,injection,compare_subset,human_labels}.jsonl`, `evals/golden/README.md`
- Acceptance:
  1. Sizes as in spec section 6: 100 retrieval, 100 in-scope (40 policy, 30 live-data, 30 mixed; each with an expected route, at least 30 `simple_lookup`), 40 out-of-scope, 20 injection, 40 comparison ids, 30 human-label slots for the director.
  2. Every live-data reference answer is computed from the `SEED=42` dataset with the query used to compute it recorded.
  3. Every retrieval item lists expected chunks by `(document filename, heading)` so ids survive re-chunking.
  4. A schema test validates every line; no real company or customer data.

#### CP-34: Integration tests for tenancy, pagination and authorization across all endpoints
- Phase 2 (P2) | Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-21, CP-22, CP-23, CP-24, CP-27, CP-47
- Files: as in overlap check
- Acceptance:
  1. `test_cross_tenant_access_returns_404_for_every_endpoint` iterates every route in the bundled contract.
  2. `test_every_list_route_is_paginated` checks every list route rejects `limit=101` and returns stable pages.
  3. Writes a target-vs-actual table with pass/fail per standard to `reports/standards/`.
  4. Any bug found is filed as an issue, not fixed.

#### CP-35: Web typed API client and Sites screens
- Phase 2 (P2) | Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-11, CP-21, CP-22
- Files: as in overlap check
- Acceptance:
  1. Generated types from the bundled contract; a thin client adds auth header, `Idempotency-Key` on writes and handles problem+json and 429 `Retry-After`.
  2. Sites list (cursor Next/Previous), site detail with rooms, room detail with racks; admin can create/edit/archive.
  3. Playwright test covers list, paging, create and an error message for 409.
  4. Pages render in under **1 s** locally on seeded data.

#### CP-36: Planner agent tools (read-only, tenant-scoped)
- MVP (M6) | Owner: builder | Risk: risk:high (AI agent loop) | Size: ~600 lines | Depends on: CP-18, CP-20, CP-23, CP-24, CP-30, CP-32
- Files: as in overlap check
- Acceptance:
  1. Tools `search_policies`, `get_site_capacity`, `get_utilization`, `get_forecast`, `list_reservations` with Pydantic argument schemas and JSON-schema definitions usable by both Haiku and Sonnet; none accepts `tenant_id`.
  2. Each tool calls existing services only (capacity, utilization, reservations, forecasts, retrieval; no new SQL), runs under the caller's tenant, returns at most **4,000 characters** wrapped by `wrap_untrusted`.
  3. `test_agent_tools_cannot_read_other_tenant` passes for every tool.
  4. Each tool has a **3 s** timeout and returns a structured error result instead of raising.

#### CP-37: Evaluation harness and `make eval`
- MVP (M6) | Owner: qa | Risk: risk:low (tests only) | Size: ~750 lines | Depends on: CP-30, CP-31, CP-33
- Files: as in overlap check
- Acceptance:
  1. `make eval` seeds `cp_eval` with `SEED=42`, runs retrieval metrics now and planner metrics when `cp_agent.answer` exists, and prints a table of every target in spec section 6 with actual and pass/fail, overall and per route.
  2. Reports p95 time to first token, p95 total time, and mean and p95 cost per question in dollars split by model, from a dated price table in `evals/config/prices.yaml`; judge and Opus comparison costs reported separately.
  3. Judge calls use `CP_EVAL_JUDGE_MODEL` at temperature 0 with `max_tokens` 512; judge agreement with `human_labels.jsonl` reported once labels exist.
  4. Opus comparison re-runs `compare_subset.jsonl` with `CP_EVAL_COMPARE_MODEL` as the full-path model and reports correctness, faithfulness, p95 total time and cost differences (report only).
  5. Hard spend cap of **$10.00** per run across live path, judge and Opus comparison; the run stops and fails if it would exceed it. `make eval EVAL_SUBSET=smoke` runs 20 questions without the comparison for under **$1**.
  6. Writes `reports/eval/<timestamp>.md` and `.json`; exits non-zero if any gated target fails; `test_make_eval_reports_quality_latency_cost` passes.

#### CP-38: Web reservation screens
- Phase 2 (P3) | Owner: builder | Risk: risk:medium | Size: ~650 lines | Depends on: CP-23, CP-35
- Files: as in overlap check
- Acceptance:
  1. Request form (site, room, optional rack, kW, RU, dates, justification) with client-side validation matching the API.
  2. List filtered by status with cursor paging; detail shows history and allocations.
  3. Approver view with approve, reject, allocate (auto or choose racks), release; buttons shown only for allowed roles and hidden on one's own requests.
  4. `capacity-exceeded` and `invalid-transition` errors shown in plain English; Playwright covers the full request -> approve -> allocate path.

#### CP-39: Web forecast screens
- Phase 2 (P3) | Owner: builder | Risk: risk:medium | Size: ~500 lines | Depends on: CP-24, CP-28, CP-29, CP-35
- Files: as in overlap check
- Acceptance:
  1. 90-day forecast chart with 80% and 95% bands, committed reservations overlaid, and capacity line.
  2. Each chart and number names its metric from docs/metrics.md in a tooltip.
  3. Shows "Not enough history" for a site with under 56 days of data.
  4. Playwright test checks the chart renders for a seeded site and the empty state for a new site.

#### CP-40: Load tests for API SLOs and ingest
- Phase 2 (P3) | Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-20, CP-21, CP-22, CP-23, CP-24, CP-29
- Files: as in overlap check
- Acceptance:
  1. `make load` runs the API mix at **50 requests/second** for 10 min (1 min warm-up) and reports read p95, write p95 and 5xx rate vs targets.
  2. Ingest scenario at **2,000 points/second** for 15 min reports throughput, 202 p95, freshness p95/p99, 429 count, and loss = accepted - stored - dead-lettered.
  3. Burst scenario at **4,000 points/second** for 2 min proves 429s occur, the cap is never exceeded and the queue drains below 5,000 within 60 s.
  4. Each scenario is run 3 times and the median reported in `reports/load/` with pass/fail per row.

#### CP-41: CI hardening
- Phase 2 (P1) | Owner: release | Risk: risk:high (CI/CD) | Size: ~300 lines | Depends on: CP-15
- Files: as in overlap check
- Acceptance:
  1. CI installs uv and pnpm (pinned, checksum-verified), runs `make check` and `make test` with Testcontainers, and builds images for arm64 and amd64.
  2. `check-actions-pinned.sh` fails the build if any action is not pinned to a 40-character SHA.
  3. CI generates an SBOM and runs Trivy on built images (zero unfixed HIGH/CRITICAL).
  4. Dependabot enabled for pip/uv, npm, Docker and GitHub Actions; every job has `timeout-minutes`.

#### CP-42: Partition maintenance, retention and batched rollup backfill
- Phase 2 (P1) | Owner: data-engineer | Risk: risk:high (deleting data) | Size: ~450 lines | Depends on: CP-16, CP-20
- Files: as in overlap check
- Acceptance:
  1. Daily job creates telemetry partitions **14 days** ahead; retention job drops raw partitions older than **14 days** (approved Q6) and deletes hourly rollups older than **400 days** and daily rollups older than **5 years**, in batches of 5,000. Partition create and drop go only through `SECURITY DEFINER` functions owned by `cp_owner` with a fixed `search_path` (`cp_create_telemetry_partitions` and `cp_drop_telemetry_partitions_before`, both created in CP-13's migration 0002, so this ticket adds no migration). The daily create step may run as `cp_app` (`EXECUTE` on the create function only); the retention drop step runs as `cp_scheduler`, the only role allowed to `EXECUTE` the drop function. Neither role is granted DDL rights.
  2. `backfill_rollups` recomputes rollups for a date range in keyset batches of **5,000** rows, checkpointing in `job_checkpoints`; `test_backfill_is_batched_and_resumable` passes.
  3. Retention never drops a partition newer than today minus 14 days (`test_retention_function_refuses_cutoff_inside_14_days`, which also proves `cp_app` cannot call the drop function and `cp_scheduler` can); dry-run mode lists what would be dropped.
  4. Each job logs rows affected and duration; PR shows `EXPLAIN` for the batch query.

#### CP-43: Planner agent loop: Haiku router, Haiku and Sonnet answer paths, caps, cost guard, streaming
- MVP (M7) | Owner: builder | Risk: risk:high (AI agent loop) | Size: ~750 lines | Depends on: CP-09, CP-32, CP-36, CP-37
- Files: as in overlap check
- Acceptance:
  1. `answer(ctx, question)` calls the router (`CP_AGENT_ROUTER_MODEL`, `max_tokens` 64, 2 s timeout); routes `out_of_scope` to a fixed polite refusal, `simple_lookup` to Haiku (max 3 tool calls), `full` (and any router failure) to Sonnet (`CP_AGENT_ANSWER_MODEL`); streams `tool`, `token`, `citation`, `done` events via the resilience client, with prompt caching on both models.
  2. `registry.py` is the explicit tool allow-list; `test_agent_stops_after_8_tool_calls` (counted across all models), `test_agent_hard_timeout_15s`, `test_every_llm_call_sets_max_tokens` (64 router / 512 intermediate / 1,024 final), `test_cost_guard_stops_before_limit` ($0.02 total, path budgets in section 6) and `test_opus_not_in_live_path` pass.
  3. Answers cite document chunks and tool results; route and models used are returned in `done`.
  4. `test_trace_continues_into_llm_call` passes; spans record route, model, tokens and cost in dollars; prompts with user data are not logged.
  5. PR includes a full `make eval` table (including the Opus comparison) meeting every gated target in spec section 6, or states the gap and stops for director review.

#### CP-44: Real authentication provider (waits for a provider choice, Q13)
- Phase 2 (P2) | Owner: security | Risk: risk:high (auth, dependencies) | Size: ~600 lines | Depends on: CP-12; director choice of provider
- Files: as in overlap check
- Acceptance:
  1. Implements `AuthProvider` for the chosen provider (token validation, signing-key rotation, expiry, audience, issuer).
  2. Maps provider identities and groups to `tenant_id` and the roles in spec section 2.
  3. Failed logins back off per user and IP (security rule); logins and permission changes audit-logged.
  4. All existing authorization tests pass unchanged with the new provider in test mode; threat model updated.
  5. Migration `0003_users_memberships.py` adds `users` and `memberships` as tenant-owned tables (forced RLS, composite tenant keys), expand-only and reversible; the tenancy and migration named tests cover them, and the PR shows upgrade and downgrade output.

#### CP-45: Failure-injection tests
- Phase 2 (P1) | Owner: qa | Risk: risk:low (tests only) | Size: ~500 lines | Depends on: CP-19, CP-20, CP-23, CP-27, CP-43
- Files: as in overlap check
- Acceptance:
  1. Redis stopped during ingest -> 503 + `Retry-After`, no 5xx storm, recovery within **30 s** of Redis returning.
  2. Worker killed mid-batch -> items reclaimed after 60 s; no loss, no duplicates.
  3. Postgres connection drop during reservation approval -> no half-applied state (event, reservation and outbox agree).
  4. Claude returns 429/500 or hangs on the router or answer model -> retries with backoff, router failure falls back to the full path, and a graceful error event arrives within 15 s.
  5. Results table with pass/fail per scenario in `reports/failure/`.

#### CP-46: Web documents screen
- Phase 2 (P3) | Owner: builder | Risk: risk:medium | Size: ~400 lines | Depends on: CP-27, CP-35
- Files: as in overlap check
- Acceptance:
  1. Admin uploads `.md`/`.txt` up to 2 MB, sees status move from pending to embedded or failed.
  2. List with cursor paging; delete with confirmation.
  3. Non-admins see the list read-only.
  4. Playwright covers upload, status refresh and delete.

#### CP-47: Planner question API (Server-Sent Events) and history
- MVP (M8) | Owner: builder | Risk: risk:high (AI agent loop) | Size: ~550 lines | Depends on: CP-12, CP-17, CP-43
- Files: as in overlap check
- Acceptance:
  1. `POST /v1/planner/questions` streams SSE events from `answer(...)`; first event `meta` includes `question_id`.
  2. Concurrency bounded at **20** in-flight questions per API instance and **5** per tenant; over the limit -> 429 + `Retry-After`; `test_planner_returns_429_when_concurrency_limit_reached` passes.
  3. Per-user limit **10 questions/minute** and per-tenant AI spend cap **$10/day** (approved Q9) -> 429 when exceeded.
  4. Each question stored in `planner_questions` with route, models used, status, citations, tool calls, tokens, cost in dollars and latency; list and detail endpoints paginated and tenant-scoped.
  5. End-to-end p95 time to first token <= **1.5 s** and total <= **6 s** measured through the API on the smoke eval subset.
  6. Authorization tests per route including cross-tenant -> 404 and `ingestor` -> 403.
  7. Repeated `Idempotency-Key` behaves as in section 3: while the first answer is still streaming -> **409** + `Retry-After: 2`; after it finished -> **200** with the stored answer as JSON and `Idempotent-Replay: true`, with no new model calls and no added cost; after a `timed_out` or `error` attempt, or once an in-progress key is older than **75 s** (15 s hard timeout + 60 s margin, e.g. after a crash), the repeat runs again as a fresh answer. `test_planner_idempotent_replay_returns_stored_answer` and `test_in_progress_idempotency_key_expires_after_timeout_and_can_retry` pass.

#### CP-48: Nightly evaluation workflow
- Phase 2 (P2) | Owner: release | Risk: risk:high (CI/CD) | Size: ~120 lines | Depends on: CP-37, CP-41, CP-43
- Files: as in overlap check
- Acceptance:
  1. Runs `make eval` nightly at **03:00 UTC** and on manual dispatch, with `ANTHROPIC_API_KEY` and `CP_EMBED_API_KEY` from a GitHub environment secret.
  2. Uploads `reports/eval/` as an artifact; fails the job if any gated target fails.
  3. `timeout-minutes: 60`; actions pinned to SHAs; spend per run under **$10** including the Opus comparison (from the report).

#### CP-49: Staging deployment (DEFERRED until after MVP, Q7)
- Phase 2 (P3, deferred) | Owner: release | Risk: risk:high (infra, CI/CD) | Size: ~500 lines | Depends on: CP-26, CP-41, CP-44; staging host is the director's home K3s cluster (Raspberry Pi, arm64; Q14)
- Files: as in overlap check
- Acceptance:
  1. Deploys images to the chosen staging host only after director approval via the GitHub `staging` environment.
  2. Secrets only from GitHub environments; no credentials printed.
  3. Runs migrations as `cp_owner`, then smoke tests `/readyz` and one authenticated read.
  4. Rollback to the previous image documented and tested once; availability SLO for staging set and measured.

#### CP-50: Web planner chat screen
- Phase 2 (P3) | Owner: builder | Risk: risk:medium | Size: ~500 lines | Depends on: CP-11, CP-21, CP-22, CP-26, CP-35, CP-47, CP-53
- Files: as in overlap check
- Acceptance:
  1. Chat input (<= 2,000 characters) streams the answer token by token, shows tool steps ("Checking DAL-1 forecast...") and clickable citations.
  2. Shows refusals, time-outs, budget stops and 429 "busy, try again in N s" in plain English.
  3. History list of past questions with cursor paging.
  4. Playwright test with a mocked SSE stream covers streaming, citation click and error display.

#### CP-51: Final acceptance report
- Phase 2 (P4) | Owner: qa | Risk: risk:low (tests only) | Size: ~300 lines | Depends on: all MVP and Phase 2 tickets except CP-44 and CP-49 if still waiting
- Files: as in overlap check
- Acceptance:
  1. One table with every target in spec sections 4, 5 and 6 and every named test in section 7: target, actual, pass/fail.
  2. Runs on a clean `make down && make up && make seed` from scratch.
  3. Lists every open issue labelled `found-by-agent` with severity.

#### CP-52: Pre-launch security audit (OWASP ASVS Level 1)
- Phase 2 (P4) | Owner: security | Risk: risk:high (security agent rule) | Size: 0 lines of code | Depends on: CP-26, CP-47
- Files: none (opens GitHub issues only)
- Acceptance:
  1. Checks the repository against ASVS Level 1 and docs/threat-model.md.
  2. Opens one issue per gap with `bug`, `s1|s2|s3` and risk labels; "S1 FOUND" reported to the director immediately if any.
  3. Summary of gaps by severity returned to the director in plain English.

#### CP-53: Web container image and compose web service (new, split from CP-15)
- Phase 2 (P2) | Owner: release | Risk: risk:high (infra) | Size: ~150 lines | Depends on: CP-11, CP-15
- Files: as in overlap check
- Acceptance:
  1. Web image builds for linux/arm64 and linux/amd64; base image pinned by digest.
  2. Runs as non-root with read-only root filesystem and all capabilities dropped; `make check-images` covers it.
  3. `make up` serves the web app on host port **18080**, healthy within **90 s**.
  4. Trivy finds zero unfixed HIGH or CRITICAL issues.

#### CP-54: Site capacity panel on the Sites screen (new, split from the first-draft CP-39)
- Phase 2 (P4) | Owner: builder | Risk: risk:medium | Size: ~300 lines | Depends on: CP-29, CP-35
- Files: as in overlap check
- Acceptance:
  1. Per-room table of capacity, committed, used and headroom for kW and RU, with headroom under **20%** highlighted (approved Q11; matches the capacity planning guideline).
  2. Each number names its metric from docs/metrics.md in a tooltip.
  3. Playwright test checks the table for a seeded site and the highlight on a nearly full room.

### 9.4 Ticket summary

| Phase | Wave | Tickets | Owners |
|---|---|---|---|
| MVP | M1 | CP-01, CP-02, CP-03, CP-04 | release, builder, security, analytics |
| MVP | M2 | CP-05, CP-06, CP-07, CP-08, CP-09, CP-10 | data-engineer, builder x4, analytics |
| MVP | M3 | CP-12, CP-13, CP-14, CP-15, CP-16, CP-17, CP-18 | security, data-engineer x2, release, builder x2, analytics |
| MVP | M4 | CP-19, CP-20, CP-23, CP-24, CP-25, CP-27 | builder x4, data-engineer x2 |
| MVP | M5 | CP-28, CP-30, CP-31, CP-32, CP-33 | analytics, builder, data-engineer, security, qa |
| MVP | M6 | CP-36, CP-37 | builder, qa |
| MVP | M7 | CP-43 | builder |
| MVP | M8 | CP-47 | builder |
| Phase 2 | P1 | CP-11, CP-21, CP-22, CP-26, CP-41, CP-42, CP-45 | builder x3, security, release, data-engineer, qa |
| Phase 2 | P2 | CP-29, CP-34, CP-35, CP-44 (waiting), CP-48, CP-53 | builder x2, qa, security, release x2 |
| Phase 2 | P3 | CP-38, CP-39, CP-40, CP-46, CP-49 (deferred), CP-50 | builder x4, qa, release |
| Phase 2 | P4 | CP-51, CP-52, CP-54 | qa, security, builder |

Totals: MVP **32** tickets in **8** waves; Phase 2 **22** tickets in **4** waves; **54** tickets overall.

---

## 10. Assumptions made by the architect

- Users and roles come from the local dev login (Q1); `actor_id` is a string from the auth layer.
- Policy documents are Markdown or plain text only in v1 (PDF would add a parsing dependency).
- Embedding vector size is **1,024**, matching the chosen Voyage AI model family; the exact model name is
  configuration (`CP_EMBED_MODEL`) chosen in CP-27.
- The eval judge defaults to Sonnet 5.5 (not specified by the director); it is configurable and
  calibrated against the director's 30 labels.
- Model prices are not hard-coded in this spec; they go in a dated price table when CP-37 is built.
- Local tracing uses an in-memory exporter in tests and an optional OTLP endpoint; no trace viewer is
  added because only four host ports are allowed.
- The 2,000 points/second ingest target is headroom; the synthetic steady state is about 85 points/second.
- The release agent may only edit `.github/`, `infra/` and Dockerfiles, so the Makefile is owned by the
  builder (CP-02) and calls `infra/` scripts.
- The MVP runs only on the local laptop stack with synthetic data, which is why general rate limits and
  security headers (CP-26) and the real login (CP-44) can wait for Phase 2.

---

## 11. Decisions and open questions for the director

### Decided (2026-10-17)

| # | Question | Decision |
|---|---|---|
| Q1 | Login provider | **Local login for now** (dev provider, ADR 0008). Real provider deferred to Phase 2 (CP-44); see Q13 |
| Q2 | Claude models | **Haiku 5.5** (`claude-haiku-5-5`) routes questions and answers simple lookups; **Sonnet 5.5** (`claude-sonnet-5-5`) writes full answers; **Opus 5.5** (`claude-opus-5-5`) is compared in `make eval` only, never in live requests (ADR 0007) |
| Q3 | Embedding vendor | **Voyage AI**, 1,024 dimensions |
| Q4 | Forecast accuracy targets | **Approved** as in section 6 |
| Q5 | Reservation approvals | `approver` and `admin` roles approve for their whole tenant; **nobody approves their own request**; approved but unallocated reservations **expire after 14 days**; no site-level approvers in this version |
| Q6 | Telemetry retention | Raw **14 days**, hourly **400 days**, daily **5 years** |
| Q7 | Staging host | **Deferred until after the MVP** (CP-49 in Phase 2); host later chosen in Q14: the home K3s cluster. Reason: the director's rule was "use the proposal if it is local (Colima) or free, otherwise defer". My draft did not make a single proposal; it listed examples (a cloud account, a managed container service, the home K3s cluster). None of them is the local Colima stack, and the cloud options are not free, so the "local or free" condition is not met and staging is deferred. The availability SLO is deferred with it |
| Q8 | Synthetic policy documents | The 6 proposed documents (section 1) |
| Q9 | AI spending limits | **$10/day** per tenant for the live assistant; **$10** per full eval run, now including the Opus comparison (section 6) |
| Q10 | Human labels for the AI judge | The **director** labels the 30 calibration answers (about 1 hour), using a form QA provides |
| Q11 | Other proposed targets | **All approved**: forecast freshness by 02:30 UTC, document searchable within 120 s, 20% low-headroom highlight, false refusal <= 0.05, citation presence >= 0.95, judge reliability >= 0.85, rate limits (50 requests/second per tenant, 20 per IP), 14-day reservation expiry, 14-day raw retention, spend caps, 15 s agent timeout |
| Q12 | Screen in the MVP? | **No.** The MVP stays API-only (32 tickets, section 9.1 accepted as laid out). The planner chat screen (CP-50) is the first screen built in Phase 2, right after CP-26 (rate limits and security headers); see "Phase 2 order" in section 9.2 |
| Q13 | Real login provider (CP-44) | **Decision point: the start of Phase 2.** The director chooses the provider before any Phase 2 work that leaves the laptop; CP-44 and CP-49 cannot start until then |
| Q14 | Phase 2 staging host (CP-49) | **The director's home K3s cluster (Raspberry Pi, arm64).** Images already build for arm64 and amd64. The staging availability SLO is set when CP-49 is built |
| ADRs | ADR 0001 to 0008 | **All Accepted on 2026-10-17, approved by the director** |
| PR size | This pull request is about 1,900 changed lines (over the ~800 guideline) | **Exception approved by the director** because it is docs only (Confirmed by the director in a PR comment on PR #5; the director's merge of this PR is the final record) |

### Still open

Nothing blocks the MVP. Open items for later: the login provider choice (Q13, at the start of Phase 2)
and the staging availability SLO number (set with CP-49).
