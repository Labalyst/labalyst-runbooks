# Runbook: Backend 5xx burst

> **Alert class(es)**: `backend.5xx_burst` (P1, ≥3% for 5 min),
> `backend.5xx_elevated` (P2, 1-3% for 10 min),
> `backend.error_logs` (P2, >20 ERROR/min for 5 min),
> `infra.latency.p95` (P2 >2s / P1 >5s for 10 min).
> **Tier(s)**: P1 (burst, severe latency) / P2 (elevated, error_logs, moderate latency).
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

The `production-backend` Cloud Run service is returning HTTP 5xx
responses at an elevated rate (or logging ERROR-severity messages at
high volume). This is usually one of three things: a deploy regression,
a downstream dependency (Cloud SQL, Redis) going sideways, or a runtime
resource exhaustion (memory, CPU, connection pool). Real users are
seeing error pages.

## First-response checks (in order)

1. **Check Cloud SQL and Redis zonal health.** Before anything else,
   rule out infrastructure. We run a single-zone Cloud SQL + Redis
   deployment today; a zone event cascades into backend 5xx.
   ```bash
   # Cloud SQL up? (instance name carries a generated suffix)
   gcloud sql instances list --project=production-482716 \
     --filter='name~^production-postgres-' \
     --format='value(name,state,region,settings.availabilityType)'
   # Redis up?
   gcloud redis instances describe production-redis \
     --region=us-central1 --project=production-482716 \
     --format='value(state,host,port,locationId)'
   # GCP zonal status
   curl -fsS https://status.cloud.google.com/incidents.json \
     | jq '.[] | select(.affected_products[]?.title | test("Cloud Run|SQL|Redis|Memorystore"))
                  | {name, status_impact, begin, end}'
   ```
   If either is down, this runbook stops here — follow
   `db-connection-pool-exhaustion.md` (DB) or escalate to
   `@platform-sre` (Redis / GCP).

2. **Pull the 5xx sample — identify the endpoint and the revision.**
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND httpRequest.status>=500' \
     --project=production-482716 --limit=100 --format=json \
     > /tmp/5xx.json
   # Top endpoints
   jq -r '.[].httpRequest.requestUrl' /tmp/5xx.json \
     | sed 's|\?.*||' | sort | uniq -c | sort -rn | head -10
   # Top revisions
   jq -r '.[].resource.labels.revision_name' /tmp/5xx.json \
     | sort | uniq -c | sort -rn
   ```
   If one revision dominates the 5xx count, go to step 4 (rollback).
   If multiple revisions are implicated, the cause is environmental;
   continue at step 3.

3. **Correlate with the most recent deploy.** Cloud Run deploy events
   are logged as Cloud Audit events:
   ```bash
   gcloud run revisions list --service=production-backend \
     --region=us-central1 --project=production-482716 \
     --limit=5 --format='table(name,creationTimestamp,active)'
   ```
   If the latest `active=True` revision was deployed within 15 min of
   the 5xx spike onset, assume deploy-attributable and roll back
   (step 4). Otherwise look at ERROR logs for stack traces:
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND severity>=ERROR
      AND NOT jsonPayload.message=~"health"' \
     --project=production-482716 --limit=50 --format=json \
     | jq -r '.[].jsonPayload | .logger + ": " + .message' \
     | sort | uniq -c | sort -rn | head -10
   ```

4. **Rollback procedure (use when step 2 or 3 points to a single revision).**
   ```bash
   # List last 5 revisions with traffic share
   gcloud run services describe production-backend \
     --region=us-central1 --project=production-482716 \
     --format='table(status.traffic[].revisionName,status.traffic[].percent)'
   # Revert all traffic to the previous known-good revision
   gcloud run services update-traffic production-backend \
     --region=us-central1 --project=production-482716 \
     --to-revisions=production-backend-<PREVIOUS-GOOD>=100
   # Verify
   gcloud run services describe production-backend \
     --region=us-central1 --project=production-482716 \
     --format='value(status.traffic[].percent,status.traffic[].revisionName)'
   ```
   Expect 5xx rate to drop within 60 s as LB routing converges.

5. **If no rollback candidate, look for resource pressure.** Cloud Run
   memory/CPU pressure:
   ```bash
   gcloud monitoring time-series list \
     --project=production-482716 \
     --filter='metric.type="run.googleapis.com/container/memory/utilizations"
              AND resource.labels.service_name="production-backend"' \
     --interval-end-time=$(date -u -d '5 min ago' +%Y-%m-%dT%H:%M:%SZ)
   ```
   Also check DB pool usage (see `db-connection-pool-exhaustion.md`
   first-response step 1).

## How to confirm it is resolved

- Cloud Run 5xx rate <1% for 10 continuous minutes.
- `api-health` uptime check passes from all 5 regions.
- No new `backend.5xx_*` alerts fire within the cool-down window.
- Cloud Run dashboard shows request-count back within the normal band
  (https://console.cloud.google.com/run/detail/us-central1/production-backend/metrics).

## When to escalate or ask for help

- If rollback does not clear the 5xx within 5 minutes, escalate to
  `@platform-lead` for code-level triage.
- If the stack traces reference `psycopg`/`sqlalchemy` pool errors,
  switch to `db-connection-pool-exhaustion.md`.
- If Cloud SQL or Redis shows non-green state, escalate to
  `@platform-sre` and reference GCP status page (Cloud SQL, Memorystore
  are the typical culprits in a zonal event; this runbook's step 1 note
  about single-zone deployment is tracked as follow-up issue #2147).
- Bring to the escalation: the 5xx sample `/tmp/5xx.json`, the list of
  revisions with traffic share, and the most recent ERROR log cluster.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-R1, AC-R2, AC-DS-1, AC-DS-2.
- Signal sources: `backend.5xx_burst`, `backend.5xx_elevated`,
  `backend.error_logs`, `infra.latency.p95` — policies in
  `terraform/modules/monitoring/production-alerts/backend/`.
- Related runbooks: `db-connection-pool-exhaustion.md`,
  `uptime-check-failure.md`.
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
