# labalyst-runbooks

Operational runbooks for the **Labalyst production alerting** pipeline
(Epic #2103). This repo is **public-readable** and **independent of every
system the runbooks describe** — so a responder can open a runbook from
the alert on their phone even when login, the main app, or the GCP
project itself is degraded.

- **Primary URL shape**: `https://github.com/labalyst/labalyst-runbooks/blob/main/<slug>.md`
- **Fallback URL shape** (nightly GCS mirror): `https://ops.labalyst.ai/runbooks/<slug>.html`

Every Cloud Monitoring alert policy in production carries both a
`runbook_url` and `runbook_url_fallback` label that resolve to one of
the slugs below (or to this README, per the default-runbook policy
section).

---

## What to do if you got here from an alert

If your phone just paged you and you tapped the runbook link in the
payload, you are either here because:

1. **The alert's `signal_class` has no dedicated runbook slug yet** —
   the default-runbook policy points uncovered signals at this README.
   Read the next two paragraphs.
2. **You misclicked or the URL was wrong** — the day-1 slug index is
   below.

### Immediate first move (uncovered signal)

- If the signal **looks like alerting-itself failing** (heartbeat
  absent, synthetic test unacked, hc.io down), open
  [alerting-pipeline-broken.md](./alerting-pipeline-broken.md). That
  runbook is the meta-runbook for "the alerting pipeline is the
  thing that's broken."
- Otherwise, **post in the `ops-alerts` Google Chat space** describing
  the alert, then open the GitHub issue named in the payload's
  `policy_id` label — the issue body has the signal-specific context
  the architect/SRE wrote when the policy was created.
- **Escalation**: if no other responder acks within 15 minutes, follow
  the escalation steps in
  [recipient-lifecycle.md](./recipient-lifecycle.md).

A signal landing here means the platform lead owes a follow-up: either
write a slug-specific runbook, or update the alert policy's
`runbook_url` to point at an existing one. Open an issue against this
repo so it doesn't get lost.

---

## Day-1 runbook index

| Slug                                                                           | Covers                                                                                              |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| [`login-auth-outage.md`](./login-auth-outage.md)                               | Auth/login envelope — 6 signal classes + composite (Clerk, error rate, latency, token-refresh, etc.) |
| [`backend-5xx-burst.md`](./backend-5xx-burst.md)                               | Backend 5xx bursts and elevated p95 latency                                                         |
| [`db-connection-pool-exhaustion.md`](./db-connection-pool-exhaustion.md)       | Postgres pool/CPU/disk/replica-lag                                                                  |
| [`uptime-check-failure.md`](./uptime-check-failure.md)                         | External uptime probe failures                                                                      |
| [`cost-spike.md`](./cost-spike.md)                                             | Cost runaway, budget thresholds, AI-token runaway                                                   |
| [`alerting-pipeline-broken.md`](./alerting-pipeline-broken.md)                 | Heartbeat absent (internal/external), synthetic-P1 unacked                                          |

Plus operational artifacts (not alert-targeted, but referenced from the
runbooks above):

- [`recipient-lifecycle.md`](./recipient-lifecycle.md) — enrollment,
  offboarding, monthly hygiene checklists.
- [`recipient-roster.md`](./recipient-roster.md) — append-only log of
  enrolled responders. Stores **Pushover group keys** (Secret Manager
  refs), never user keys, never personal contact info.
- [`gcp-alerting-best-practices.md`](./gcp-alerting-best-practices.md) —
  standalone reference doc on coverage patterns, signal sources,
  channels, alerting-on-alerting, payload conventions.

---

## Page template — required structure for any new runbook

Every runbook in this repo uses the exact five-section structure below.
This is a hard requirement: payload rendering, the GCS mirror's HTML
template, and reviewer checklists all assume it.

````markdown
# Runbook: {title}

> **Alert class(es)**: `{signal_class_a}`, `{signal_class_b}`
> **Tier(s)**: P1 / P2
> **Last reviewed**: YYYY-MM-DD by {platform-lead}

## What this alert means

1–3 short paragraphs. Define jargon ("5xx burst", "pool exhaustion") if
needed, briefly. No links to internal Labalyst URLs (responder may not
be authenticated).

## First-response checks (in order)

1. **{Check name}** — {one-line action} — {expected healthy state}

   ```bash
   # optional concrete copyable command
   ```

2. **{Check name}** — ...
3. **{Check name}** — ...

## How to confirm it is resolved

- {Observable signal that says the incident is over}
- {How long to wait before declaring "resolved"}

## When to escalate or ask for help

- {Condition that means "stop trying alone"}
- {Who to reach, on what channel}
- {What information to bring}

## Related ACs, signal sources, last-reviewed date

- PRD AC references: {AC-IDs}
- Signal sources: {Cloud Monitoring policy IDs that point here}
- Last reviewed: YYYY-MM-DD
- Next review due: YYYY-MM-DD (quarterly)
````

**Visual conventions** (UX §6.3):

- One H1 per file (`# Runbook: {title}`); each section is H2.
- Numbered first-response list — order is load-bearing.
- Fenced code blocks for any copyable command, with language hint.
- No images. No collapsed `<details>` blocks. No links to Labalyst
  product URLs (AC-R3: must not depend on any system this epic
  alerts on). Links to Cloud Monitoring are fine.

---

## Quarterly review cadence (AC-R5)

- **Owner**: platform lead.
- **Cadence**: quarterly, on calendar-quarter boundary.
- **Scope**: every slug referenced by an active Cloud Monitoring alert
  policy must (a) HEAD 200 on both primary + fallback URLs and (b)
  match current signal semantics. Update `Last reviewed` and `Next
  review due` in each file's metadata blockquote.

The six **day-1 slugs** above may not be removed without
product-owner sign-off — each corresponds to a live Cloud Monitoring
policy whose payload references the slug via Terraform `runbook_url`.
Additions are welcome when a new signal class is introduced; follow
the template above and register the slug in the corresponding alert
policy in the same PR.

---

## Contributing

### Secret-scan CI

Every PR runs `gitleaks` via `.github/workflows/secret-scan.yml`. The
workflow uses the upstream default ruleset plus a repo-local
`.gitleaks.toml` extending the rules with patterns for shapes that
could plausibly leak from this repo:

- Pushover **user keys** (`u[a-z0-9]{29}`)
- Pushover **app tokens** (`a[a-z0-9]{29}`)
- Pushover **group keys** (`g[a-z0-9]{29}`)
- healthchecks.io ping URLs (`https://hc-ping.com/<uuid>`)
- Google Chat webhook URLs (`https://chat.googleapis.com/v1/spaces/...`)
- Clerk secret keys (`sk_(test|live)_...`)
- GCP service-account JSON (`"private_key": "-----BEGIN ...`)
- Tenant UUIDs (PRD §9 — any UUID not pre-allowlisted)

Branch protection on `main` requires the workflow to pass — no merge
on a failing scan.

**False positives**: add a justified entry to `.gitleaks-allowlist`
(committed in the same PR), and request platform-lead review on that
PR. No self-approval on allowlist additions.

### What MUST NOT be in this repo

- Real secrets, API keys, tokens (covered by the scan above).
- Tenant data — UUIDs, names, emails, lab content.
- Individual contact info — use group/role channels only
  (`recipient-roster.md` stores Secret Manager **refs** to Pushover
  group keys, never the keys themselves and never user keys).
- Internal hostnames beyond `ops.labalyst.ai`.
- Source code or architecture internals beyond what a runbook needs.

### Style

- Phone-first: <80-character lines for body text where possible (the
  GCS mirror styles to ≥16px base font; long lines wrap awkwardly on
  iOS Safari at 320px).
- ASCII only. No images, no embedded SVG.
- Markdown tables are fine; keep them <4 columns for phone rendering.

---

## How alert payloads reference these runbooks

For reference (not editable from this repo) — Cloud Monitoring policies
in `labalyst/labalyst` set:

```hcl
labels = {
  runbook_url          = "https://github.com/labalyst/labalyst-runbooks/blob/main/<slug>.md"
  runbook_url_fallback = "https://ops.labalyst.ai/runbooks/<slug>.html"
}
```

Both URLs end up in every outbound Pushover/email/Google Chat payload
via `/ops/alert-dispatch`. The fallback is the GCS mirror that renders
this repo nightly via Cloud Build into
`gs://labi-runbook-mirror/runbooks/`.

If a policy has no signal-specific slug, both labels point at this
README (Gap 2 default-runbook policy). See
`scripts/alerts/mark-fp.sh` in `labalyst/labalyst` for the
responder-side false-positive tagging tool.
