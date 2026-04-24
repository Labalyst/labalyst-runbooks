# Runbook: Cost spike

> **Alert class(es)**: `cost.runaway` (P1, rate >5× 7d median AND >$5/hr
> for 10 min), `cost.budget_threshold` (P1, ≥75% of monthly budget),
> `cost.ai_tokens_runaway` (P1, >5× trailing AND >100 000 tokens/5min
> for 15 min).
> **Tier(s)**: P1.
> **Last reviewed**: 2026-04-22 by @platform-lead

## What this alert means

Spend velocity has jumped far above our 7-day baseline, OR we are
approaching a monthly budget ceiling, OR AI-token consumption is
spiking. Left unchecked this can burn the monthly budget in hours. P1
because a sustained runaway is unbounded — it keeps charging until
killed.

Three sub-classes; choose response by `signal_class` on the payload:
- `cost.runaway` — generic GCP spend spike (Cloud Run, Cloud SQL,
  egress, Vertex AI, etc.).
- `cost.budget_threshold` — slower (monthly-budget) signal, still P1
  so platform lead sees it.
- `cost.ai_tokens_runaway` — specifically AI tokens metered via
  `labi_metering/ai_tokens_total_5min`.

## First-response checks (in order)

1. **Kill the runaway via feature flag first.** For
   `cost.ai_tokens_runaway` and the common `cost.runaway` shape (LLM
   calls dominating), the fastest mitigation is the LLM feature flag
   kill-switch. This cuts AI-feature traffic without taking the site
   down.
   ```bash
   # Current flag state
   gcloud run services describe production-backend \
     --region=us-central1 --project=labalyst-prod \
     --format='value(spec.template.spec.containers[0].env)' \
     | tr ',' '\n' | grep -E 'AI_|VOICE_|FEATURE_'
   # Flip: set AI_ENABLED=false (adjust variable name to match current IaC)
   gcloud run services update production-backend \
     --region=us-central1 --project=labalyst-prod \
     --update-env-vars=AI_ENABLED=false
   ```
   Expect `ai_tokens_total_5min` to flatten within 2-3 minutes.

2. **Identify the top-N spenders.** For AI-token spikes, the payload's
   `tenant_context.tenant_ids` enrichment carries the top-3 tenants by
   5-min spend. Cross-reference with `custom.googleapis.com/
   labi_metering/ai_tokens_by_feature` — which feature is driving?
   ```bash
   gcloud monitoring time-series list \
     --project=labalyst-prod \
     --filter='metric.type="custom.googleapis.com/labi_metering/ai_tokens_by_feature"' \
     --interval-start-time=$(date -u -d '15 min ago' +%Y-%m-%dT%H:%M:%SZ) \
     --format='value(metric.labels.feature,points[0].value.doubleValue)' \
     | sort -k2 -rn | head -10
   ```
   For generic `cost.runaway`, open the per-project billing breakdown:
   ```
   https://console.cloud.google.com/billing/<billing-account-id>/reports
     ?project=labalyst-prod
   ```
   Filter by service, group by SKU, time window "last 6 hours". The
   top row is your culprit.

3. **Check for a runaway Celery worker.** A stuck or looping Celery
   task can spin compute and AI-API calls indefinitely.
   ```bash
   gcloud run services describe production-celery \
     --region=us-central1 --project=labalyst-prod \
     --format='value(status.traffic[].revisionName,spec.template.spec.containers[0].resources)'
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-celery"
      AND jsonPayload.event=~"task_(retry|failed|published)"' \
     --project=labalyst-prod --limit=100 --format=json \
     | jq -r '.[].jsonPayload.task_name' | sort | uniq -c | sort -rn | head
   ```
   If one task name dominates retries, stop the worker revision
   (scale to zero) or roll back to a known-good revision. Coordinate
   with `@platform-backend` before scaling worker to zero.

4. **Apply a per-tenant Redis rate-limit (AI-token specific).** When
   one or two tenants drive the spike but the feature must stay on for
   everyone else, push a per-tenant cap via Redis — the rate-limit
   key path is `labi:ratelimit:ai_tokens:<tenant_id>`. Example SET:
   ```bash
   redis-cli -h "$LABI_PROD_REDIS_HOST" -p 6379 \
     SET labi:ratelimit:ai_tokens:<tenant_id> "1000" EX 3600
   ```
   The backend honors this within one request cycle. Contact the
   tenant via the support channel documented in
   `recipient-lifecycle.md` before or immediately after limiting.

5. **Budget-threshold specifics (`cost.budget_threshold`).** Slower
   clock — you have 60+ min, not 5. Actions: (a) pull the monthly
   burndown from the billing report above, (b) coordinate with
   product-finance on whether to lift the budget cap or throttle,
   (c) file a retrospective issue with the week's spend breakdown for
   post-incident review.

## How to confirm it is resolved

- `labi_metering/ai_tokens_total_5min` drops below 2× trailing median
  for 15 continuous minutes.
- GCP billing "last hour" spend returns to baseline band.
- No new `cost.*` policies fire within the cool-down window (30 min).
- Feature flag kill-switches have been un-flipped or scheduled for
  un-flip with a rollback-plan note in the incident.

## When to escalate or ask for help

- If the runaway cannot be traced to a feature flag, a tenant, a
  Celery task, or a deploy — escalate to `@platform-lead` and
  `@platform-finops` immediately.
- If `cost.budget_threshold` is firing alongside `cost.runaway`, this
  is post-cap territory — loop in product-finance.
- If AI-token rate-limit actions affect >10 tenants, this is a
  product-communication event — loop in customer support.
- Bring to the escalation: the top-N tenant list, the billing report
  screenshot, the feature flag state before/after, and any rate-limit
  Redis keys set.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-R1, AC-R2, AC-C8 (AI-token anomaly), SM-1
  (cost MTTD), AC-DS-2 (deploy-attributable flag).
- Signal sources: `cost.runaway`, `cost.budget_threshold`,
  `cost.ai_tokens_runaway` — policies in
  `terraform/modules/monitoring/production-alerts/cost/`.
- Related docs: architecture §2.5 (cost MTTD), §2.6 (AI-token
  decision), adr-063 (AI-token metering).
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead)
