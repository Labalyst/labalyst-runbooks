# Runbook: Alerting pipeline broken

> **Alert class(es)**: `meta.heartbeat_absent_internal` (F-6, Cloud
> Monitoring absent-metric),
> `meta.heartbeat_absent_external` (F-4, healthchecks.io external
> watchdog), `synthetic.p1_test_unacked` (VAL-1 weekly synthetic P1
> test).
> **Tier(s)**: P1.
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

The alerting *system itself* has a problem. One of:

- **Internal heartbeat absent** (`meta.heartbeat_absent_internal`):
  Cloud Scheduler `heartbeat-internal` should hit
  `/ops/heartbeat` every 60 s, which writes
  `custom.googleapis.com/labi_alerting/heartbeat_ok`. Cloud Monitoring
  `condition_absent` on that metric for >6 min means either ops-service
  is down, Cloud Scheduler is blocked, or the evaluator itself stalled.
- **External heartbeat absent** (`meta.heartbeat_absent_external`):
  healthchecks.io was expected to see a ping every 60 s and did not.
  This signal fires out-of-GCP — if you got the alert, Cloud Monitoring
  is probably fine; ops-service or network path is not.
- **Both absent**: assume a widespread outage. You are in a meta
  incident — real user-facing signals may also be suppressed.
- **Synthetic P1 unacked** (`synthetic.p1_test_unacked`): the
  weekly synthetic test fired but no on-call human acked within the
  SLA. Could be a delivery problem (Pushover) OR a responder problem.

## First-response checks (in order)

1. **Which heartbeat fired?** Read the `signal_class` on the payload.
   Pick the branch:
   - `meta.heartbeat_absent_internal` → go to step 2.
   - `meta.heartbeat_absent_external` → go to step 3.
   - BOTH fired within 5 min of each other → step 4 (widespread).
   - `synthetic.p1_test_unacked` → step 5.

2. **Internal heartbeat absent — diagnostic tree.**
   - (a) **Cloud Monitoring evaluator stalled?** If `heartbeat-external`
     also fired around the same time, the evaluator is probably fine.
     If `heartbeat-external` did NOT fire but the internal one did,
     the evaluator is suspect — check Cloud Monitoring status:
     ```bash
     curl -fsS https://status.cloud.google.com/incidents.json \
       | jq '.[] | select(.end == null and (.affected_products[]?.title | test("Monitoring")))'
     ```
   - (b) **ops-service down?** Test the heartbeat endpoint directly:
     ```bash
     gcloud run services describe ops-service \
       --region=us-central1 --project=labalyst-prod \
       --format='yaml(status.conditions,status.latestReadyRevisionName)'
     curl -fsS -m 5 https://ops.labalyst.ai/ops/heartbeat | jq .
     ```
     Expect HTTP 200 + `{"status":"ok"}`. If 5xx or timeout, ops-service
     itself is the problem — roll back its revision via the same
     procedure as `backend-5xx-burst.md` step 4 (swap
     `production-backend` for `ops-service`).
   - (c) **Cloud Scheduler job stuck?** Verify the job is enabled and
     its last-run status:
     ```bash
     gcloud scheduler jobs describe heartbeat-internal \
       --location=us-central1 --project=labalyst-prod \
       --format='yaml(state,lastAttemptTime,status)'
     ```
     If `state != ENABLED`, enable it:
     ```bash
     gcloud scheduler jobs resume heartbeat-internal \
       --location=us-central1 --project=labalyst-prod
     ```
   - (d) **Metric-writer SA permissions?** ops-service needs
     `roles/monitoring.metricWriter`. If a recent IAM change stripped
     it, heartbeats will silently stop.
     ```bash
     gcloud projects get-iam-policy labalyst-prod \
       --flatten='bindings[].members' \
       --filter='bindings.role:roles/monitoring.metricWriter' \
       --format='value(bindings.members)'
     ```
     Look for the ops-service runtime service account.

3. **External heartbeat absent — diagnostic tree.**
   - (a) **healthchecks.io itself?** Check hc.io status:
     ```bash
     curl -fsSI https://healthchecks.io/ | head -1
     ```
     If hc.io is down, the outage is vendor-attributable — the
     internal heartbeat signal is sufficient while hc.io recovers.
   - (b) **Cloud Scheduler job `heartbeat-external`?**
     ```bash
     gcloud scheduler jobs describe heartbeat-external \
       --location=us-central1 --project=labalyst-prod \
       --format='yaml(state,lastAttemptTime,status)'
     ```
     If it ran but returned non-2xx, check the ping-URL secret —
     it may have been rotated and not re-loaded:
     ```bash
     gcloud secrets versions list ops-service-hcio-ping-url \
       --project=labalyst-prod --limit=3 \
       --format='table(name,state,createTime)'
     ```
     If a newer version exists than what ops-service loaded, restart
     ops-service (deploy a no-op revision).
   - (c) **Egress path broken?** ops-service reaches hc.io over public
     internet. If Cloud NAT / egress has an issue:
     ```bash
     gcloud run services describe ops-service \
       --region=us-central1 --project=labalyst-prod \
       --format='value(spec.template.metadata.annotations)' \
       | tr ',' '\n' | grep vpc-access
     ```
     Egress via VPC connector should not require opening holes for
     public hc.io — confirm the connector exists and is healthy.

4. **Both absent — widespread incident branch.** If both internal and
   external heartbeats fired within 5 min of each other, assume a
   production-wide outage. You cannot trust Cloud Monitoring to show
   you other signals. Actions:
   1. Check `status.cloud.google.com` — if GCP region-wide outage
      in `us-central1`, wait on vendor.
   2. Open Cloud Run console manually (UI works even when metrics
      don't) to verify `production-backend` and `ops-service` are
      both running.
   3. If GCP is green and services are up, the problem is in the
      alerting pipeline's metric-write path — escalate to
      `@platform-sre` immediately.
   4. DO NOT `mark-fp` any incidents while both heartbeats are down.
      Use `scripts/alerts/mark-fp.sh` only after the pipeline is
      confirmed healthy (see step 6).

5. **Synthetic P1 unacked — `synthetic.p1_test_unacked`.**
   Distinguishes "delivery broken" from "responder missed it":
   - (a) Pull the Pushover delivery receipt for the synthetic:
     ```bash
     # The weekly synthetic logs its delivery_id to Cloud Logging:
     gcloud logging read \
       'resource.type="cloud_run_revision"
        AND resource.labels.service_name="ops-service"
        AND jsonPayload.event="synthetic_p1_dispatched"' \
       --project=labalyst-prod --limit=5 --format=json \
       | jq -r '.[] | "\(.timestamp) delivery_id=\(.jsonPayload.pushover_delivery_id) status=\(.jsonPayload.pushover_status)"'
     ```
   - (b) If Pushover delivery_id exists but `pushover_status != 1`,
     Pushover had a send error — go to step 7 (Pushover CLI test).
   - (c) If delivery_id exists and status=1, Pushover delivered but no
     human acked — not a pipeline problem; follow up with the on-call
     responder per `recipient-lifecycle.md` monthly-hygiene step 3.
   - (d) Also verify email fallback arrived — ask the responder or
     check the fallback mailbox.

6. **When to `mark-fp` — hold off until pipeline is healthy.** This
   runbook does NOT recommend marking the alert False Positive while
   the pipeline's own signal integrity is in doubt. Once you have
   confirmed:
   - Cloud Monitoring is green,
   - ops-service is up and `/ops/heartbeat` returns 200,
   - Cloud Scheduler heartbeat jobs are `ENABLED` and green,
   then and only then, if the incident you are investigating was a
   genuine alerting glitch (e.g., a known Cloud Monitoring
   evaluator hiccup), mark it:
   ```bash
   ./scripts/alerts/mark-fp.sh <incident-id> \
     --reason "cloud monitoring evaluator stalled for ~6 min; \
               internal heartbeat absent but all other signals green"
   ```
   See `scripts/alerts/mark-fp.sh --help` for the full flag list.

7. **Manual Pushover CLI test (Pushover-down diagnosis).** If you
   suspect Pushover itself is down or our token is invalid, exercise
   it directly from any shell (adjust credentials paths):
   ```bash
   # Pull secrets (local, short-lived)
   PUSHOVER_TOKEN=$(gcloud secrets versions access latest \
     --secret=pushover-app-token --project=labalyst-prod)
   PUSHOVER_GROUP=$(gcloud secrets versions access latest \
     --secret=pushover-group-key --project=labalyst-prod)
   # Send a manual P1-class test
   curl -fsS https://api.pushover.net/1/messages.json \
     -F "token=$PUSHOVER_TOKEN" \
     -F "user=$PUSHOVER_GROUP" \
     -F "priority=2" -F "expire=300" -F "retry=60" \
     -F "title=ALERT PIPELINE MANUAL TEST" \
     -F "message=Diagnostic test from alerting-pipeline-broken.md" \
     | jq .
   ```
   Expect `{"status":1,"request":"..."}`. Anything else = Pushover-side
   problem. Status page: `https://status.pushover.net`.

## How to confirm it is resolved

- Both heartbeats (internal AND external) return green for 10
  continuous minutes.
- `/ops/heartbeat` returns 200 from at least two separate shells.
- Cloud Scheduler `heartbeat-internal` and `heartbeat-external` last
  attempts both report `SUCCEEDED` within the last 2 minutes.
- A manual synthetic P1 trigger (per `recipient-lifecycle.md`
  monthly-hygiene step 2) delivers on both Pushover and the email
  fallback and is acked.

## When to escalate or ask for help

- If both heartbeats are absent simultaneously and GCP is green,
  escalate to `@platform-sre` immediately — this is a pipeline-wide
  failure and user-facing signals may be silently suppressed.
- If Pushover CLI test fails, escalate to `@platform-lead` and prepare
  to temporarily route P1s through the email fallback only (Pushover
  is the primary path; email is the secondary per AC-AoA-4).
- If Cloud Monitoring itself is in an impaired state per GCP status,
  wait on vendor ETA but keep watch on the external heartbeat — it
  is out-of-GCP and remains reliable.
- Bring to the escalation: the outputs of steps 2(b), 2(c), and 3(b)
  as applicable; any recent IAM changes; a Pushover-CLI test result.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-AoA-1, AC-AoA-2, AC-AoA-3, AC-AoA-5,
  AC-R1, AC-R2.
- Signal sources: `meta.heartbeat_absent_internal`,
  `meta.heartbeat_absent_external`, `synthetic.p1_test_unacked` —
  policies in `terraform/modules/monitoring/heartbeat/` and
  `terraform/modules/monitoring/production-alerts/security/`.
- Related scripts: `scripts/alerts/mark-fp.sh` (use only after
  pipeline is confirmed healthy — see step 6).
- Related runbooks: `recipient-lifecycle.md` (monthly-hygiene).
- External status pages:
  `https://status.cloud.google.com/`,
  `https://status.pushover.net/`,
  `https://healthchecks.io/`.
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
