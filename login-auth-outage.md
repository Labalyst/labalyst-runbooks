# Runbook: Login / auth outage

> **Alert class(es)**: Architecture §2.7 six-class envelope —
> class 1 (`auth.error_rate`), class 2 (`auth.latency_p95`),
> class 3 (`auth.clerk_health`), class 4 (`auth.login_rate_drop`),
> class 5 (`auth.token_refresh`), class 6 (`auth.synthetic_login`).
> Plus composite upgrade `auth.envelope_p2_cluster` when ≥2 classes fire
> within 10 min.
> **Tier(s)**: P1 (classes 1, 3, 4, 6 individually; classes 2, 5 when sustained;
> composite cluster = P1).
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

A meaningful share of real users cannot sign in or resume an authenticated
session. This is the PRD §6.3 six-class envelope: the alert that fired is
one signal among six that together cover "login is broken" regardless of
shape (burst, silent drop, IDP issue, token-refresh loop, synthetic probe
fail). Users are locked out. This is P1.

The envelope is not a single condition — it is six independent checks over
the login funnel. Each class has its own first-response paragraph below.

## First-response checks (in order)

1. **Identify which envelope class fired.** Read the `Class:` /
   `Signal class:` line of the payload. Map it to a class number using
   this table and jump to the matching class paragraph below.

   | Envelope class | `signal_class` | Topic |
   |---|---|---|
   | Class 1 | `auth.error_rate` | Auth endpoint 5xx ratio |
   | Class 2 | `auth.latency_p95` | Auth endpoint latency |
   | Class 3 | `auth.clerk_health` | Upstream IDP (Clerk) health |
   | Class 4 | `auth.login_rate_drop` | Successful-login rate drop |
   | Class 5 | `auth.token_refresh` | Token-refresh failure loop |
   | Class 6 | `auth.synthetic_login` | External synthetic probe |
   | Composite | `auth.envelope_p2_cluster` | ≥2 classes in 10 min |

2. **Class 1 — `auth.error_rate` simple burst.** Classic backend
   regression shape. Usually correlates with a recent Cloud Run revision.
   Pull the last 50 5xx-severity events and see if they're clustered on a
   single revision:
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND httpRequest.requestUrl=~"/api/v1/auth/"
      AND severity>=ERROR' \
     --project=production-482716 --limit=50 --format=json \
     | jq -r '.[].resource.labels.revision_name' | sort | uniq -c | sort -rn
   ```
   If one revision dominates, roll back (see class-2 command block).

3. **Class 2 — `auth.latency_p95` / upstream IDP latency.** Auth endpoints
   are slow but not returning 5xx. Usually upstream Clerk latency. Check
   Clerk status first, then backend log volume:
   ```bash
   curl -fsS https://status.clerk.com/api/v2/status.json | jq .status
   ```
   If Clerk reports `none`, check backend logs for slow Clerk calls:
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND jsonPayload.logger="app.auth.clerk_client"' \
     --project=production-482716 --limit=100 --format='value(jsonPayload.duration_ms)' \
     | awk '{s+=$1; n++} END {if (n>0) print "mean="s/n"ms n="n}'
   ```
   Rollback procedure:
   ```bash
   # Identify the last-known-good revision
   gcloud run revisions list --service=production-backend \
     --region=us-central1 --project=production-482716 --limit=5
   # Route 100% traffic to that revision
   gcloud run services update-traffic production-backend \
     --region=us-central1 --project=production-482716 \
     --to-revisions=production-backend-<GOOD-REVISION>=100
   ```

4. **Class 3 — `auth.clerk_health` / upstream IDP outage.** The Clerk
   status page reports a non-`none` incident, OR the
   `clerk-status` uptime check failed 3-of-5 regions. This is
   not-a-thing-we-can-fix-locally. Verify:
   ```bash
   curl -fsS https://status.clerk.com/api/v2/summary.json \
     | jq '.incidents[] | {name, status, impact, created_at}'
   ```
   Post the Clerk incident link in the `ops-alerts` Google Chat space
   and skip to "When to escalate". Suppress new pages for the duration
   via a short-window `muting_rules` entry if the Clerk incident is
   expected to last >30 min.

5. **Class 4 — `auth.login_rate_drop` / silent session-create collapse.**
   `successful_logins_per_minute` dropped below the 15-min P1 window
   baseline. No elevated 5xx (that would have been class 1). The funnel
   is broken somewhere between the auth endpoint and session creation.
   Check the session-create code path in backend logs:
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND jsonPayload.logger="app.auth.session_service"
      AND jsonPayload.event=~"session_(created|create_failed)"' \
     --project=production-482716 --limit=200 --format=json \
     | jq -r '.[].jsonPayload.event' | sort | uniq -c
   ```
   Also check the Postgres `users` / `sessions` table for a write-lock
   (catalog-level):
   ```bash
   psql "$LABI_PROD_PSQL" -c \
     "SELECT pid, state, wait_event_type, query
        FROM pg_stat_activity
        WHERE query ILIKE '%sessions%' AND state != 'idle';"
   ```

6. **Class 5 — `auth.token_refresh` / token-refresh loop.** Clients are
   refreshing tokens but failing, driving a loop. Look for
   `token_refresh_failed` events:
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND jsonPayload.event="token_refresh_failed"' \
     --project=production-482716 --limit=50 --format=json \
     | jq -r '.[].jsonPayload.reason' | sort | uniq -c | sort -rn
   ```
   Top reasons: `clerk_jwt_expired`, `clock_skew`, `revoked_session`.
   If dominated by `clerk_jwt_expired`, this is a JWT-key rotation event
   — confirm with Clerk dashboard and wait for the next refresh cycle.

7. **Class 6 — `auth.synthetic_login` / external synthetic probe fail.**
   The `synthetic-login` Cloud Monitoring uptime check failed 3-of-5
   regions for 2 consecutive evaluations. This is the most reliable
   black-box signal. Open the synthetic dashboard from the alert's
   `incident_url`:
   ```
   https://console.cloud.google.com/monitoring/uptime
   ```
   and inspect which regions failed and with what HTTP status. Common
   causes: (a) synthetic-login credential expired — regenerate via the
   enrollment checklist in `recipient-lifecycle.md`; (b) Cloud Run
   `/ops/synthetic-login` endpoint down — check Cloud Run service
   health; (c) real outage — classes 1-5 will likely also fire shortly
   (composite upgrade to `auth.envelope_p2_cluster`).

## How to confirm it is resolved

- The `signal_class` that fired returns below threshold for 10
  continuous minutes (e.g., class 1 `auth.error_rate` drops below 1%).
- `auth.synthetic_login` returns 3-of-3 success across the 5 regions.
- No new envelope-class alerts fire in the 10 minutes after the original
  ack.
- Dashboard "Alert Health — Rolling 30 Days" shows no new P1 fires
  against this policy family within the cool-down window.

## When to escalate or ask for help

- If the alert is still firing 30 minutes after acknowledgement, post
  in the Google Chat `ops-alerts` space with the `signal_class`,
  `incident_url`, and the `policy_id` from the payload.
- If Clerk shows a status-page incident (class 3), post the Clerk
  incident link in `ops-alerts` — no local fix path.
- If Cloud Monitoring itself is unreachable (cannot open
  `incident_url`), follow `alerting-pipeline-broken.md` — you are in a
  meta-outage.
- Escalation contact: `@platform-lead` for auth-domain rollbacks;
  `@platform-sre` for infra; `@platform-finops` only if a cost signal
  also fires.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-LI-1, AC-LI-2, AC-LI-3, AC-LI-5, AC-R1, AC-R2.
- Signal sources: Cloud Monitoring policies implementing architecture
  §2.7 Decision 7 — six classes (`auth.error_rate`, `auth.latency_p95`,
  `auth.clerk_health`, `auth.login_rate_drop`, `auth.token_refresh`,
  `auth.synthetic_login`) plus composite `auth.envelope_p2_cluster`.
  Policy IDs are resolved at `terraform apply`; each alert payload
  carries its own `policy_id`.
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
