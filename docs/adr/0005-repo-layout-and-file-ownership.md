# ADR 0005: Repository layout and file ownership (so tickets can run in parallel)

- Status: Proposed (needs director approval)
- Date: 2026-10-17
- Related: docs/spec.md "Tickets by wave"

## Decision in one line

The code is split into small, clearly named folders, and the few files everyone would normally edit
(the app's start-up file, the route list, the settings file, the dependency lock file, the Makefile)
are designed so new features **add new files** instead of editing shared ones. That is what lets
several agents work at the same time without stepping on each other.

## Context

- The director wants waves of tickets where **no two tickets in a wave touch the same file**.
- Normal web projects have "hot" files: `main.py` (registering routes), one settings file, one OpenAPI
  file, one lock file, one Alembic chain, one navigation menu.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| One Python package, conventional layout | Familiar | Every feature edits `main.py`, settings, the OpenAPI file: constant conflicts |
| **Workspace of small packages + auto-discovery + split contract** (chosen) | Features are additive files; parallel-safe | A little up-front plumbing; rules must be followed |
| Separate repositories per service | Total isolation | Overkill; cross-repo changes are painful for one director to approve |

## Decision

Layout (Python packages use `uv` workspace members; web uses pnpm + Turborepo):

```
Makefile                      targets delegate to discovered packages (written once, CP-02)
.env.example                  names only, every planned variable listed up front (CP-01)
pyproject.toml, uv.lock       uv workspace root
infra/compose/compose.yaml    local stack; infra/docker/*.Dockerfile; infra/postgres/init/; infra/redis/
contracts/openapi/            base.yaml + paths/<resource>.yaml + components/<resource>.yaml
core/                         shared library: db, tenancy, pagination, streams, resilience, services
  core/migrations/            Alembic (single linear chain)
api/                          FastAPI app; api/src/cp_api/routers/<resource>.py, plugins/<name>.py
worker/                       consumers/<stream>.py, schedules/<job>.py, jobs/<backfill>.py
forecast/                     model, baseline, backtest, nightly job
agent/                        planner agent loop, tools/<tool>.py, retrieval, guards
seed/                         synthetic data generator and loaders; seed/policies/*.md
evals/                        golden/ (datasets), src/ (harness), config/ (prices, thresholds)
tests/                        cross-service integration and failure tests (QA)
load/                         load tests (QA)
reports/                      eval, load and backtest result tables
apps/web/                     Next.js app; one folder per screen under apps/web/app/<section>/
docs/                         spec, ADRs, threat model, metrics, lessons
```

Parallel-safety rules:
1. **Routers and plugins are auto-discovered**: the API imports every module in
   `cp_api/routers/` and `cp_api/plugins/` at start-up. A new endpoint is a new file.
2. **Worker consumers, schedules and jobs are auto-discovered** the same way.
3. **Models are auto-imported** by `cp_core/db/models/__init__.py` (written once) so Alembic sees new
   model files without editing a shared list.
4. **Settings are per module**: each module defines its own small Pydantic settings class with its own
   environment prefix (`CP_STREAMS_`, `CP_AGENT_`, ...). No shared settings file grows.
5. **OpenAPI 3.1 contract is split by resource** and bundled by a script into
   `contracts/openapi/dist/openapi.json` (generated, not committed). A contract test checks the running
   app matches the bundle.
6. **Dependencies are front-loaded**: the baseline ticket (CP-02 for Python, CP-11 for web) adds every
   dependency we can foresee. Any later ticket that must add a dependency (and so edits `uv.lock` or
   `pnpm-lock.yaml`) must be the only ticket in its wave touching that lock file, and says why in the PR.
7. **One Alembic migration per wave** (ADR 0004).
8. **Shared docs**: `docs/threat-model.md` is edited by at most one security ticket per wave;
   `docs/metrics.md` by at most one analytics ticket per wave.
9. **Web navigation** is created once with all planned sections (placeholder pages); each screen
   ticket replaces only its own `apps/web/app/<section>/` folder.
10. Changed-line estimates in tickets **exclude** lock files and generated files.

## Consequences

- Good: waves can run fully in parallel; merge conflicts are rare and small.
- Cost: auto-discovery is a little "magic"; a test lists discovered routes so nothing is silently missed
  (`test_route_inventory_matches_contract`).
- Cost: front-loaded dependencies sit unused for a few waves; the trade-off is acceptable.
