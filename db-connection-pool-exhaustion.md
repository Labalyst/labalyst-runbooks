# Runbook: DB connection pool exhaustion

> **Alert class(es)**: `infra.db.pool` (P1, >70% `num_backends` for 5 min),
> `infra.db.cpu` (P2, >85% for 10 min),
> `infra.db.disk` (P2, >85% for 10 min),
> `infra.db.replica_lag` (P2, >60 s for 10 min).
> **Tier(s)**: P1 (pool) / P2 (cpu, disk, replica_lag).
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

The Cloud SQL Postgres instance for `production-db` is at or near its
connection pool ceiling. Postgres `max_connections` is a hard ceiling;
when we cross ~70% the backend starts seeing `OperationalError:
connection pool is full` and cascades into 5xx. This is almost always
one of: a long-running / stuck query, a connection leak in application
code (missing `session.close()` / `commit()`), or a genuine traffic
spike that legitimately needs more pool headroom.

Replica lag, CPU, and disk are sibling P2 conditions — often precursors
or consequences of the same root cause.

## First-response checks (in order)

1. **Count active sessions and their state.** Opens a direct psql to
   the prod instance via the Cloud SQL proxy.
   ```bash
   gcloud sql connect production-db --user=postgres --project=labalyst-prod
   -- Inside psql:
   SELECT state, count(*) FROM pg_stat_activity
    WHERE datname='labi_db' GROUP BY state ORDER BY count(*) DESC;
   SELECT setting::int AS max_connections FROM pg_settings
    WHERE name='max_connections';
   ```
   If `active + idle in transaction > 0.7 * max_connections`, the
   signal is confirmed.

2. **Find the top long-running queries.** Long-running queries are the
   single most common trigger.
   ```sql
   SELECT
     pid,
     age(clock_timestamp(), query_start) AS duration,
     state,
     wait_event_type,
     wait_event,
     left(query, 200) AS query_head
   FROM pg_stat_activity
   WHERE state != 'idle'
     AND query NOT LIKE '%pg_stat_activity%'
   ORDER BY duration DESC
   LIMIT 20;
   ```
   Flag any session with `duration > 60 s` AND `state = 'active'`. For
   those, decide: kill, or escalate to the engineer who shipped the
   query. If the query is obviously pathological (full-table scan on a
   large table, missing `LIMIT`), kill:
   ```sql
   SELECT pg_terminate_backend(<pid>);  -- only after review
   ```

3. **Look for connection-leak signatures.** Leaks appear as long-lived
   `idle in transaction` sessions from backend pods:
   ```sql
   SELECT
     pid,
     application_name,
     client_addr,
     state,
     age(clock_timestamp(), state_change) AS idle_for,
     left(query, 200) AS last_query
   FROM pg_stat_activity
   WHERE state = 'idle in transaction'
     AND age(clock_timestamp(), state_change) > interval '2 minutes'
   ORDER BY idle_for DESC;
   ```
   Cluster by `application_name` (SQLAlchemy sets this) — all from one
   app-version? Likely a code leak; roll back the revision per
   `backend-5xx-burst.md` step 4.

4. **Open Cloud SQL Insights for query-level hot spots.**
   ```
   https://console.cloud.google.com/sql/instances/production-db/insights
     ?project=labalyst-prod
   ```
   Sort by "Total time" over the last hour. Missing indexes and
   sequential scans show up here.

5. **Emergency pool resize (use only after steps 1-4 fail to clear).**
   Raising `max_connections` is the nuclear option — it costs RAM on
   the Cloud SQL VM and sidesteps the real cause. Only do this if you
   are actively losing users and cannot find a rollback candidate.
   ```bash
   # Current setting
   gcloud sql instances describe production-db --project=labalyst-prod \
     --format='value(settings.databaseFlags)'
   # Bump (example: 200 -> 300). This triggers a brief restart.
   gcloud sql instances patch production-db --project=labalyst-prod \
     --database-flags=max_connections=300
   ```
   Record the change in an incident note; file a follow-up to reduce
   once the underlying leak is fixed.

## How to confirm it is resolved

- `pg_stat_activity` count drops below `0.5 * max_connections` for 10
  continuous minutes.
- `cloudsql.googleapis.com/database/postgresql/num_backends` metric
  returns to its 7-day baseline.
- Backend Cloud Run revision no longer logs `connection pool is full`
  or `QueuePool limit ... overflow` stack traces.
- `infra.db.cpu` and `infra.db.replica_lag` return to green (P2
  siblings often clear together).

## When to escalate or ask for help

- If active session count does not drop after terminating the top 3
  long-running queries, escalate to `@platform-lead` and
  `@platform-sre`.
- If a runaway query is identified but cannot be safely killed (e.g.
  mid-migration), loop in the on-call engineer for the shipping team.
- If the alert is coupled with `backend.5xx_burst`, the user-visible
  outage takes priority — roll back the suspect backend revision first
  (see `backend-5xx-burst.md`), then return here.
- Bring to the escalation: the output of steps 1-3 (table of sessions,
  top long-running queries, leak-signature cluster) and any recent
  deploy timestamp.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-R1, AC-R2.
- Signal sources: `infra.db.pool`, `infra.db.cpu`, `infra.db.disk`,
  `infra.db.replica_lag` — policies in
  `terraform/modules/monitoring/production-alerts/db-redis/`.
- Related runbooks: `backend-5xx-burst.md`.
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
