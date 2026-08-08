# Runbook: WAF configuration drift

> **Alert class(es)**: `security.waf_config_drift`
> **Tier(s)**: P1
> **Last reviewed**: 2026-08-08 by {platform-lead}

## What this alert means

The hourly WAF reconciliation check found that the live Cloud Armor
configuration differs from the enforced Terraform authority. It checks policy
attachment on all edge backends, request logging, the managed-rule floor, and
whether a WAF rule returned to preview after the enforcement deadline.

Read the alert summary first; it names the failed check types. The notification
is intentionally compact. Read the `waf-reconcile: drift detected` ERROR entry
in the `waf-config-reconcile` function logs for every affected resource and
the complete details before changing production.

## First-response checks (in order)

1. **Identify the failed check.** Map each payload item to one of:

   - `policy_attachment`: a backend is detached or points at another policy.
   - `log_config_disabled`: a backend stopped emitting Cloud Armor logs.
   - `rule_count_drop`: one or more Terraform-managed rules disappeared.
   - `preview_past_deadline`: a rule observes traffic but no longer enforces.

2. **Read the live policy and backend state.** Run:

   ```bash
   gcloud compute security-policies describe edge-block-policy \
     --project=<PROJECT_ID> --global \
     --format='table(rules.priority,rules.action,rules.preview)'

   for service in edge-block-tenant-backend edge-block-admin-backend edge-block-frontend-backend; do
     gcloud compute backend-services describe "$service" \
       --project=<PROJECT_ID> --global \
       --format='table(name,securityPolicy,logConfig.enable)'
   done
   ```

3. **Determine whether this was a deliberate rollback.** A responder may have
   intentionally returned a rule to preview to stop false-positive blocking.
   Confirm in `ops-alerts` before restoring enforcement.

4. **Restore through Terraform.** If the drift was not deliberate, run the
   production provision workflow from the known-good main revision. Stop if
   its plan contains unrelated deletions.

5. **Re-run reconciliation through Cloud Scheduler.** The function accepts
   internal traffic only, so trigger its authenticated scheduler job instead
   of invoking the function URL from a laptop:

   ```bash
   gcloud scheduler jobs run waf-config-reconcile \
     --location=<REGION> --project=<PROJECT_ID>
   ```

   Confirm the resulting `waf-config-reconcile` function log reports
   `status: ok`; do not treat the Scheduler command being accepted as proof
   that reconciliation passed.

## How to confirm it is resolved

- The manual reconciliation result is `status: ok` with zero drift items.
- All three backends reference `edge-block-policy` and have logging enabled.
- The live managed-rule count meets the Terraform-produced floor.
- No rule is in preview unless an active incident explicitly requires it.
- The next scheduled reconciliation run is also clean.

## When to escalate or ask for help

- Escalate immediately for policy detachment or an unexplained rule deletion.
- Escalate before reversing a deliberate preview rollback; restoring a known
  false-positive rule can create a production outage.
- Escalate if Terraform cannot reproduce the expected state or proposes
  unrelated deletions.
- Post in `ops-alerts`, tag `@platform-lead`, and bring the complete
  `drift_items` object from the function ERROR log, live policy output, and
  Terraform plan.

## Related ACs, signal sources, last-reviewed date

- Decision: ADR-110 WAF-integrity monitoring, issue #2729.
- Signal source: hourly `waf-config-reconcile` Cloud Function.
- Complementary signal: `security.waf_integrity` log absence.
- Last reviewed: 2026-08-08.
- Next review due: 2026-10-26.
