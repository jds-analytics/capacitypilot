# ADR 0003: Tenancy (keeping each customer's data separate)

- Status: Accepted on 2026-10-17, approved by the director
- Date: 2026-10-17
- Related: docs/spec.md "Domain model", docs/threat-model.md, ADR 0008 (auth seam)

## Decision in one line

All customers ("tenants") share one database and one set of tables. Every row that belongs to a
customer carries a `tenant_id`. We protect it with **two locks**: (1) the application adds
`tenant_id = <current tenant>` to every query, and (2) the database itself refuses to show or change
another tenant's rows even if the application forgets (Postgres **row-level security**, "RLS").
We do **not** give each tenant its own schema or database.

Plain-English terms:
- **Tenant**: one customer organisation. Users of tenant A must never see tenant B's data.
- **Row-level security (RLS)**: a database rule attached to a table that filters rows automatically
  based on a setting for the current connection.
- **Schema-per-tenant**: a separate copy of every table for each customer.

## Context

- CLAUDE.md: "tenant_id on every tenant-owned row, enforced in every query."
- Auth is not decided yet. A swappable auth layer (ADR 0008) will supply `tenant_id` and roles for
  each request. Workers get `tenant_id` from the queue message.
- Expected scale: tens of tenants, hundreds of sites, large shared telemetry tables.
- The planner AI agent reads live data through tools; it must never reach another tenant's data,
  even under prompt injection.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| **Shared schema, tenant_id + RLS** (chosen) | One migration path; simple pooling; easy cross-tenant ops for admins; RLS is a database-level safety net | A bug in RLS policy setup affects everyone, so it needs strong tests; noisy-neighbour risk handled by rate limits |
| Shared schema, tenant_id only (no RLS) | Simplest | One missed `WHERE` clause leaks data; no second lock |
| Schema-per-tenant | Strong separation; per-tenant restore | Migrations run N times and drift; connection pool and search_path tricks; hundreds of copies of partitioned telemetry tables; pgvector indexes per tenant; poor fit for tens to hundreds of tenants |
| Database-per-tenant | Strongest isolation | Highest cost and ops burden; not justified for synthetic-data product stage |

## Decision

1. **Every tenant-owned table** has `tenant_id uuid NOT NULL` referencing `tenants(id)`, and every
   index used by queries **leads with `tenant_id`**.
2. **Child rows match their parent's tenant**: foreign keys are composite, e.g.
   `rooms (tenant_id, site_id) REFERENCES sites (tenant_id, id)`, so a room can never point at another
   tenant's site.
3. **First lock (application)**: all queries go through a repository layer that requires a
   `TenantContext`; there is no query function without one. Requests without a tenant are denied with
   **401** (not logged in), matching the spec: a tenant only comes from a valid login, so a request
   with no tenant is treated as not logged in. 403 is kept for "logged in, but your role may not do
   this".
4. **Second lock (database)**: RLS is `ENABLE`d and `FORCE`d on every tenant-owned table with
   policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)` and the same `WITH CHECK`.
   Each transaction runs `SET LOCAL app.tenant_id = '<id>'`. If the setting is missing, the cast
   fails and the query errors (deny by default).
5. **Database roles**:
   - `cp_owner`: owns tables, runs migrations only. Never used by the API or workers.
   - `cp_app`: used by API and workers. `NOBYPASSRLS`, no `CREATE`, no access to individual telemetry
     partitions (only the parent table, where policies apply). May call
     `cp_create_telemetry_partitions(days_ahead)` but **not** the drop function below.
   - `cp_scheduler`: may read the `tenants` table only, to loop over tenants for nightly jobs; it then
     opens a normal `cp_app` transaction per tenant. It is also the **only** role allowed to call
     `cp_drop_telemetry_partitions_before(cutoff)`, which refuses any cutoff newer than today minus
     **14 days** (approved raw retention).
   - Both partition functions are `SECURITY DEFINER` (run with `cp_owner`'s rights), use a fixed
     `search_path = pg_catalog, public, pg_temp`, and name every table with its schema
     (`test_retention_function_refuses_cutoff_inside_14_days`). `EXECUTE` on **both** functions is
     revoked from `PUBLIC` (PostgreSQL otherwise lets every role call a new function), then granted
     only to the roles named above (`test_app_role_cannot_create_or_drop_partitions_directly`).
6. **Not-found, not forbidden**: asking for another tenant's object by id returns **HTTP 404**, so ids
   cannot be probed.
7. **Queue messages** carry `tenant_id`; workers set it per message before touching the database.
8. **pgvector search** runs under the same RLS; we enable pgvector iterative index scans so filtering
   by tenant does not starve results (see risk R6 in the spec).
9. **Tests** (named in the spec): `test_every_tenant_table_has_tenant_id_and_forced_rls`,
   `test_rls_blocks_query_without_tenant_setting`, `test_cross_tenant_access_returns_404_for_every_endpoint`
   (one per endpoint), `test_child_cannot_reference_other_tenant_parent`, `test_app_role_cannot_bypass_rls`.

## Consequences

- Good: a forgotten filter in code still cannot leak data; the planner agent's tools inherit this.
- Good: one schema, one migration path, simple operations.
- Cost: every transaction must set `app.tenant_id`; we wrap this in one session helper.
- Cost: RLS adds a small planning overhead (measured, expected under **5 ms** per query) and requires
  care with partitioned tables and connection pooling (`SET LOCAL` is transaction-scoped, which is safe
  with pooling).
- Cost: no per-tenant restore; if a tenant needs it later we would export by `tenant_id`.
- Changing anything in this ADR's policies is **risk:high** and needs red-team review.
