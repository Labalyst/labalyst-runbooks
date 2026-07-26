# Runbook: Scanner anomaly

> **Alert class(es)**: `security.scanner_auth_failure` (P1),
> `security.edge_block_spike` (P1).
> **Tier(s)**: P1.
> **Last reviewed**: 2026-07-26 by {platform-lead}

## What this alert means

Something is probing the perimeter. Two signals point here:

`security.scanner_auth_failure` fires when a single source IP draws
more than N HTTP 401s in a 5-minute sliding window on the tenant
backend, counting only paths **outside** `/api/v1/`. `/api/v1/` is
excluded deliberately: 401s there are dominated by real users in
expired-session loops, and including them would guarantee false pages.

`security.edge_block_spike` fires when Cloud Armor's edge deny-list
blocks (`outcome=DENY`) more than B requests per minute for 5
consecutive minutes. Those are requests that never reached the
application at all.

Neither alert means a breach. The edge deny-list returns 404 and the
Clerk auth wall returns 401 - both signals are the perimeter reporting
that it is doing its job, at a volume worth a human look. Your job is
to decide which of three things this is: benign background scanning,
a gap worth closing in the deny-list, or a real incident.

## First-response checks (in order)

1. **Set your project once.** Every command below uses it.

   ```bash
   PROJECT=<production project id>   # GCP console > project picker
   ```

2. **Read the enrichment already in your payload.** Do not run any
   query yet. The page carries four fields that usually settle the
   triage on their own:

   | Field | What it tells you |
   | --- | --- |
   | `top_source_ips` | up to 5 `{ip, count}`, highest first |
   | `distinct_path_count` | how many distinct paths in the window |
   | `top_paths` | up to 5 `{path, count}`, highest first |
   | `saved_query_url` | Cloud Logging deep-link, same window |

   Check `signal_context.enrichment_status` first. If it is `ok`, the
   numbers are good. If it is `unavailable`, `timeout`, or `error`,
   the page went out un-enriched by design - skip to step 4 and
   gather the same facts by hand.

3. **Make the scanner-vs-real-user call. Path spread is the
   discriminator.** Many distinct paths from few IPs means a scanner
   sweeping a wordlist. Few distinct paths from many IPs means real
   users hitting something broken.

   | Shape | Read it as |
   | --- | --- |
   | High `distinct_path_count`, 1-2 IPs | scanner (the common case) |
   | Low `distinct_path_count`, many IPs | NOT a scanner - go to 3b |
   | Low count, 1 IP, one odd path | integration or bot, not a sweep |

   For `security.scanner_auth_failure` specifically: almost every
   real route on the tenant backend lives under `/api/v1/`, and this
   metric excludes that prefix. A fire here is therefore
   presumptively a scanner. The real-user shape to watch for is a
   client or integration hammering one non-`/api/v1/` path.

   **3b. If `top_paths` contains anything that looks like a real
   application path** - an app route, a `/assets/...` bundle, a
   `/.well-known/...` URI - stop treating this as a scanner. On
   `security.edge_block_spike` that pattern means a deny-list rule is
   matching legitimate traffic and **real users are being blocked at
   the edge**. That is a P1 outage, not a probe. Go straight to the
   escalation section and say "deny-list false positive".

4. **Only if enrichment was missing: pull the same facts by hand.**

   Top source IPs and path spread for an auth-failure page:

   ```bash
   gcloud logging read \
     'resource.type="cloud_run_revision"
      AND resource.labels.service_name="production-backend"
      AND httpRequest.status=401
      AND NOT httpRequest.requestUrl:"/api/v1/"' \
     --project="$PROJECT" --limit=1000 --freshness=1h \
     --format=json > /tmp/scanner.json

   jq -r '.[].httpRequest.remoteIp' /tmp/scanner.json \
     | sort | uniq -c | sort -rn | head -10
   jq -r '.[].httpRequest.requestUrl' /tmp/scanner.json \
     | sed 's|?.*||' | sort -u | wc -l      # distinct path count
   ```

   For an edge-block page, the DENY entries instead:

   ```bash
   gcloud logging read \
     'resource.type="http_load_balancer"
      AND jsonPayload.enforcedSecurityPolicy.outcome="DENY"' \
     --project="$PROJECT" --limit=1000 --freshness=1h \
     --format=json > /tmp/deny.json

   jq -r '.[].httpRequest.remoteIp' /tmp/deny.json \
     | sort | uniq -c | sort -rn | head -10
   jq -r '.[].httpRequest.requestUrl' /tmp/deny.json \
     | sed 's|?.*||' | sort | uniq -c | sort -rn | head -20
   ```

5. **Confirm the auth wall held. This is the question that matters
   for breach notification.** Everything above tells you someone
   knocked. This tells you whether any door opened. Run the standing
   forensic query over the alert window - it lists any scanner-shaped
   path that returned something other than 401 or 404:

   ```bash
   # Saved forensic template. Set WINDOW_START / WINDOW_END to the
   # alert's own window (RFC3339, UTC) before running.
   WINDOW_START=2026-01-01T00:00:00Z
   WINDOW_END=2026-01-01T01:00:00Z

   gcloud logging read \
     "resource.type=\"cloud_run_revision\"
      resource.labels.service_name=\"production-backend\"
      timestamp>=\"$WINDOW_START\"
      timestamp<\"$WINDOW_END\"
      httpRequest.requestUrl=~\"(?i)(\\.env|\\.git|phpinfo|info\\.php|php\\.php|wp-config|wp-json|\\.aws|config\\.(php|js|json)|aws-config|_environment|_profiler|webroot|robots\\.txt)\"
      httpRequest.status!=401
      httpRequest.status!=404" \
     --project="$PROJECT" --limit=200 --format=json
   ```

   **Zero rows is the expected result and closes the question.** Any
   row that is not `/robots.txt` returning 200 is a finding: something
   scanner-shaped got a real response. Escalate immediately and bring
   the rows.

6. **Decide, and record the decision.** One of three:

   - **Benign.** Background internet scanning, everything blocked,
     step 5 clean. Ack the page. If this exact shape has now paged
     more than twice in a week, say so when you ack - it is threshold
     tuning input, not a new incident.
   - **Extend the deny-list.** The sweep found paths we do not block
     yet and step 5 is clean. Follow the section below.
   - **Open an incident.** Step 5 returned rows, or step 3b said
     real users are being blocked, or the volume is sustained enough
     to be an availability concern.

7. **Extending the deny-list (only for the middle case).**

   **Read this before opening the PR: the Cloud Armor policy is at
   its CEL-evaluation rule quota (20, fully consumed by the current
   deny-list).** Adding a rule without a quota increase makes
   `terraform apply` fail, and that blocks every production deploy,
   not just this change. The WAF rate-limit and CRS rule sets are
   already rolled back to off for exactly this reason. If you need a
   new rule tonight, request the quota increase first, or fold the
   new pattern into an existing rule's regex rather than adding a
   rule. **Do not open a rule-adding PR and merge it at 3am.**

   The procedure, once quota is not in the way:

   1. Open a PR editing `deny-list.yaml` in the main repo. Fill in
      `rationale` and `incident_ref` - `incident_ref` is required and
      is what makes `git blame` answer "why is this rule here".
   2. Add the new paths to that rule's `positive_corpus`. CI evaluates
      every rule against its positive and negative corpora and fails
      the build on a miss either way.
   3. **Never add an `/api/v1/*` path, `/robots.txt`, `/`, or a
      `/.well-known/*` URI.** These are global negative controls; CI
      fails the build, and the reason is that blocking them takes the
      product down. If a scanner is probing under `/api/v1/`, that is
      out of this deny-list's scope - open a follow-up issue instead.
   4. On-call operator approval is the minimum review. CI posts the
      `terraform plan` diff to the PR.
   5. Merge, let CI apply, then confirm the synthetic-probe verifier
      replays the new path and sees it blocked.

## How to confirm it is resolved

- The alert policy auto-closes after 30 minutes with no new data.
  You do not need to close it by hand.
- No repeat fire of the same signal class within one hour.
- The forensic query in step 5 returns zero rows for the full window,
  extended to the point the traffic stopped.
- If you extended the deny-list: the next scheduled edge-block
  verifier run is green, and the newly added path returns 404.
- If it was a deny-list false positive: the legitimate path returns
  its normal status from an unauthenticated client, and the DENY
  count for that path is zero over a fresh 15-minute window.

## When to escalate or ask for help

- **Immediately, ahead of everything else**, if step 5 returned any
  row that is not `/robots.txt` returning 200. That is a possible
  disclosure and starts a 72-hour GDPR notification clock. Bring the
  raw rows, the window, and the requesting IPs.
- **Immediately** if step 3b showed real users being blocked by the
  deny-list. Availability incident. Bring `top_paths` and the DENY
  sample from step 4.
- If the volume is sustained for more than an hour, or is large
  enough to look like a capacity event rather than a scan.
- If you need a deny-list rule but the CEVAL quota blocks it, and the
  probing is active enough that waiting is not acceptable.
- **Where**: post in the `ops-alerts` Google Chat space and tag
  `@platform-lead`. If nobody acks within 15 minutes, follow the
  escalation ladder in `recipient-lifecycle.md`.
- **What to bring**: the alert payload verbatim (it carries the
  enrichment), `distinct_path_count`, the top IPs, the step 5 result,
  and which of the three decisions in step 6 you reached.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC2.1 (auth-failure spike), AC2.2 (edge-block
  spike), AC2.3 (payload contents), AC2.4 (this runbook). Forensic
  template in step 5 is the AC1.3 query.
- Signal sources: `scanner.auth_failure_spike`,
  `scanner.edge_block_spike` - policies in
  `terraform/modules/monitoring/production-alerts/security-scanner/`.
  Enrichment is added by `/ops/alert-dispatch`; it has a 5-second
  budget and pages go out un-enriched rather than late.
- Related runbooks: `alerting-pipeline-broken.md` (if you suspect the
  alert itself rather than the traffic), `backend-5xx-burst.md` (if
  the probing is causing errors rather than being blocked).
- Last reviewed: 2026-07-26
- Next review due: 2026-10-26 (quarterly cadence, platform lead)
