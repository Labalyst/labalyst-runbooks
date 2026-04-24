# GCP Alerting Best Practices

**AC-BP-1 (covered topics) + AC-BP-2 (reads standalone)**. This
document is intentionally self-contained: a reader should be able
to apply the patterns described here to a different production GCP
service without opening the Labalyst-specific architecture doc or
the paid-vs-free tool comparison in architecture §4. Versioned via
Git history (SM-10 "standalone and versioned").

## Coverage patterns

A production web service on GCP should be instrumented against
four golden signals plus five supporting categories:

1. **Latency** — p50/p95/p99 on critical user-facing endpoints.
   Usually served by Cloud Run / Load Balancer request latency
   metrics.
2. **Traffic** — request rate. Serves both capacity planning and
   as a denominator for error-rate alerts.
3. **Errors** — HTTP 5xx rate and 4xx anomalies; application log
   ERROR rate via Cloud Logging → log-based metric.
4. **Saturation** — CPU utilization, memory, DB connection pool,
   message queue depth, disk.
5. **External reachability** — Cloud Monitoring Uptime Checks from
   globally-distributed probe regions, outside the VPC.
6. **Dependency health** — explicit probes of upstream services
   (IDP, payment processor, AI/LLM APIs).
7. **Cost / quota** — Cloud Billing budget thresholds (Pub/Sub push)
   + per-service quota metrics.
8. **Security events** — IAM changes, Cloud Audit Logs, VPC flow log
   anomalies. Surfaced via log-based metrics.
9. **Pipeline self-monitoring** — alerting-on-alerting via
   `condition_absent` and an out-of-system watchdog.

A production service that covers 1-9 above has the full envelope;
missing any one leaves a failure mode that can go silent for hours.

## Signal sources

| Signal class | GCP-native source |
|---|---|
| HTTP error / latency / request count | `run.googleapis.com/*`, `loadbalancing.googleapis.com/*` |
| Application logs | Cloud Logging → log-based metrics |
| Database | `cloudsql.googleapis.com/*` |
| Cache | `redis.googleapis.com/*` (Memorystore) |
| Uptime (black-box) | Cloud Monitoring Uptime Checks |
| Synthetic user flows | Cloud Monitoring Synthetic Monitors |
| Cost (streaming) | `billing.googleapis.com/billing/cost` |
| Cost (budget) | Cloud Billing Budget → Pub/Sub push |
| Custom app counters | `google-cloud-monitoring` SDK → custom metrics |
| Security | Cloud Audit Logs → log-based metrics |
| Error aggregation | Cloud Error Reporting (auto-derived from logs) |

**Preference**: native GCP metrics before log-based metrics before
custom metrics. Native metrics ship with quota, dashboards, and
alerting policies for free; log-based metrics have ingestion cost;
custom metrics require an SDK and a metric-writer IAM binding.

## Notification channels

| Channel | Strengths | Weaknesses | Typical use |
|---|---|---|---|
| Email | Free; delivery log | No DND bypass; no actionable ack | P2, P3, fallback |
| Google Chat webhook | Team context; Workspace-native | No DND bypass; no ack | P2 |
| Pub/Sub | Programmatic downstream | Not a delivery mechanism alone | P1 via webhook |
| Webhook | Any HTTP target | No retry beyond vendor | P1 via Pushover or a paid tool |
| SMS | DND bypass | Not native — Twilio bridge; no ack | Fallback only |
| Paid paging tool (PagerDuty / Opsgenie) | Full DND bypass + ack | Per-seat cost | When budget allows |

**GCP's native channel catalog does not include a free
DND-bypassing push channel.** Production services needing a true
pager must route via a webhook to a vendor with Emergency push
semantics. Architecture §4 of the Labalyst epic documents one
specific selection of that vendor; the generic pattern is:
`Cloud Monitoring alert policy` → webhook notification channel → a
custom HTTP endpoint → `push vendor`.

**Two channels per P1 minimum** — one primary (push vendor) and
one fallback (email to an external domain). A single-channel P1 is
an undetected silent-failure exposure.

## Alerting on alerting

Two patterns used together — either alone leaves a hole:

1. **`condition_absent` on a heartbeat metric.** The alerting
   system itself writes a heartbeat sample every N seconds (Cloud
   Scheduler → Cloud Run → custom metric). A policy fires when the
   metric is absent for N×M seconds. GCP-native dead-man's-switch.
   Detects ops-service / application-side failure modes.
2. **External out-of-GCP watchdog.** A second Cloud Scheduler job
   pings a third-party (e.g. healthchecks.io free tier) every N
   seconds. When pings stop, the third party pages independently.
   Closes the "Cloud Monitoring evaluator itself stalled"
   circularity — a risk pattern 1 cannot detect.

Production systems SHOULD run both. When both heartbeats flap
simultaneously, responders should assume a widespread pipeline
outage (see a `alerting-pipeline-broken.md`-equivalent runbook for
the distinguishing diagnostic tree).

## Payload and labelling conventions

Alert policies attach a fixed set of labels that responders and
downstream tooling depend on:

| Label | Purpose |
|---|---|
| `severity` | P1 / P2 / P3 — routing key for notification channels. |
| `signal_class` | Stable machine-readable category (e.g. `backend.5xx_burst`). |
| `runbook_url` | Primary runbook URL (GitHub / wiki). |
| `runbook_url_fallback` | Fallback runbook URL (GCS / static mirror). |
| `env` | `production` / `staging` — partitions signals per environment. |
| `policy_id` | Back-reference to the Cloud Monitoring alert policy. |

**Single canonical payload** — every outbound alert (push, email,
chat) is assembled from the same set of labels so that every
surface presents consistent information. Downstream dispatchers
(webhook → Pushover, email fallback, chat) MUST whitelist the
fields they relay and must never leak internal labels
(`tenant_id` lists, DB PII) into vendor payloads.

**Runbook URL is load-bearing.** Every P1 or P2 policy MUST
carry a `runbook_url` pointing to a page that explains what the
alert means and what the first-response checks are. A policy
without a runbook URL is by definition a missing spec.

## Breaking down the four golden signals into policies

Start with a minimum of four policies per service:

1. P1 — 5xx burst (1-3% over 5 min).
2. P1 — p95 latency exceeds a user-visible threshold (e.g. 5 s).
3. P2 — elevated 5xx (1-3% over 10 min, below the P1 threshold).
4. P1 — uptime check failure from ≥3 of 5 regions.

Add saturation (DB pool, CPU), cost runaway, security log-based
metrics, and the two pipeline heartbeats as the service matures.

## Closing notes on scope

This document covers recommended **patterns**. Vendor-selection,
per-incident triage playbooks, and service-specific thresholds are
separate: service teams tune thresholds during a 14-day break-in
window post-production-go-live and document the results in a
review artifact — that review, not this page, is the source of
truth for specific numbers in a specific service.

**Review cadence**: quarterly. **Last reviewed**: 2026-04-22.
