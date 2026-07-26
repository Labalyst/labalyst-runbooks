# Runbook: Uptime check failure

> **Alert class(es)**: `infra.uptime` (P1, 3-of-5 regions fail for 2
> consecutive checks).
> **Tier(s)**: P1.
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

The `api-health` Cloud Monitoring uptime check failed from at least 3
of 5 globally distributed probe regions on 2 consecutive evaluations.
This is the black-box "can the world reach us?" signal. A 3-of-5 quorum
is resistant to single-region probe failures, so when this fires the
site is (almost certainly) down for a significant share of real users.

Three typical causes: (a) load-balancer / frontend routing issue, (b)
Cloud Run service is unhealthy or scaled to zero, (c) a downstream
dependency (Clerk, Cloud SQL, Redis) is taking the app with it.

## First-response checks (in order)

1. **Confirm from your own network — is the site actually down?**
   Run the same check the probe runs:
   ```bash
   curl -fsSI -o /dev/null -w '%{http_code} %{time_total}s\n' \
     https://api.labalyst.ai/api/v1/health
   curl -fsSI -o /dev/null -w '%{http_code} %{time_total}s\n' \
     https://app.labalyst.ai/
   ```
   If both return 200 from your network, you're seeing a probe-region
   path issue (rare) — skip to step 5. If either fails, continue.

2. **Check external status pages for GCP + Clerk.** The uptime check
   is often the first signal of a broader vendor outage.
   ```bash
   # GCP
   curl -fsS https://status.cloud.google.com/incidents.json \
     | jq '.[] | select(.end == null) | {name, begin, severity: .severity, affected_products}'
   # Clerk
   curl -fsS https://status.clerk.com/api/v2/summary.json \
     | jq '{status: .status.indicator, incidents: [.incidents[] | {name, status, impact}]}'
   ```
   If GCP shows Cloud Run, VPC, Load Balancer, or networking
   impairment in our region — this is a vendor outage, skip to step 6.
   If Clerk shows a non-`none` incident, auth-dependent endpoints may
   be failing (see `login-auth-outage.md`).

3. **Load balancer + Cloud Run service state.** Verify the LB and
   service are themselves reachable and configured correctly.
   ```bash
   # LB forwarding rule (frontend)
   gcloud compute forwarding-rules list \
     --project=production-482716 \
     --format='table(name,IPAddress,target,region)'
   # Cloud Run service state
   gcloud run services describe production-backend \
     --region=us-central1 --project=production-482716 \
     --format='yaml(status.conditions,status.traffic,status.latestReadyRevisionName)'
   ```
   Expect `Ready=True` and traffic pointing at a valid revision. If
   `Ready=False`, read the `status.conditions` message — usually a
   failed deploy or a scaling issue.

4. **Cloud Run logs — any mass-failure pattern right before the check
   started failing?** Use the uptime check's onset time as the anchor.
   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND (severity>=ERROR OR httpRequest.status>=500)' \
     --project=production-482716 --limit=50 --freshness=15m --format=json \
     | jq -r '.[] | "\(.timestamp) \(.httpRequest.status // .severity) \(.jsonPayload.message // .textPayload // "")"' \
     | head -20
   ```
   If dominated by 5xx from a single revision, follow
   `backend-5xx-burst.md` for rollback.

5. **If site is up from everywhere except GCP probe regions — check
   the probe configuration itself.** A recent Terraform change to the
   uptime check can silently break the probe path.
   ```bash
   gcloud monitoring uptime list-configs --project=production-482716 \
     --format='table(displayName,monitoredResource.type,httpCheck.path,period)' \
     | grep -i api-health
   ```
   Confirm the path is `/api/v1/health` and the host matches
   `api.labalyst.ai`. Recent false-positive patterns live in the
   labalyst-runbooks `false-positives.md` log.

6. **Vendor-attributable outage — confirm and mute.** If GCP or Clerk
   declared an active incident that maps to this alert, post the
   incident URL in `ops-alerts`. Optionally apply a time-bound mute
   via Terraform `muting_rules` if the vendor ETA is >30 min — do NOT
   disable the alert policy itself.

## How to confirm it is resolved

- All 5 uptime probe regions return success for 2 consecutive check
  intervals (~2 min after the fix).
- Cloud Run service `status.latestReadyRevisionName` matches the
  revision serving 100% traffic.
- `backend.5xx_*` policies are not co-firing.
- `curl -fsS https://api.labalyst.ai/api/v1/health` returns 200 from
  outside your VPN.

## When to escalate or ask for help

- If the service won't become Ready and no recent deploy can be rolled
  back, escalate to `@platform-lead` + `@platform-sre`.
- If GCP Cloud Run / Load Balancer is declared impaired in the
  incident feed, wait on vendor ETA — no local fix available. Update
  status in `ops-alerts` every 15 min.
- If Clerk is declared impaired and auth endpoints are the main
  failure mode, follow `login-auth-outage.md` class 3.
- Bring to the escalation: the `forwarding-rules list` output, Cloud
  Run `status.conditions`, and a timestamp of the last green probe.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-R1, AC-R2.
- Signal sources: `infra.uptime` — policy in
  `terraform/modules/monitoring/production-alerts/uptime-gcs/`.
- Related runbooks: `backend-5xx-burst.md`,
  `login-auth-outage.md`.
- External status pages:
  `https://status.cloud.google.com/`,
  `https://status.clerk.com/`.
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
