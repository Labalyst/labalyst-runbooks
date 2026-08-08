# Runbook: WAF integrity log absence

> **Alert class(es)**: `security.waf_integrity`
> **Tier(s)**: P1
> **Last reviewed**: 2026-08-08 by {platform-lead}

## What this alert means

Cloud Monitoring has seen no request-log metric data from the enforced
Cloud Armor policy for at least 15 minutes. This is a fail-open heuristic,
not proof that the policy detached: the cause can also be genuinely zero
traffic or delayed Cloud Logging ingestion.

The scheduled `security.waf_config_drift` check is the authoritative
configuration cross-check. Use both signals before changing production.

## First-response checks (in order)

1. **Confirm whether edge traffic exists.** In Cloud Logging, query HTTP load
   balancer requests over the alert window without filtering on a security
   policy. If there is genuinely no traffic, keep observing for 15 minutes.

2. **Confirm that Cloud Armor logs are absent.** Run:

   ```bash
   gcloud logging read \
     'resource.type="http_load_balancer"
      AND jsonPayload.enforcedSecurityPolicy.name="edge-block-policy"' \
     --project=<PROJECT_ID> --freshness=30m --limit=20
   ```

3. **Check policy attachment on every edge backend.** Run:

   ```bash
   for service in edge-block-tenant-backend edge-block-admin-backend edge-block-frontend-backend; do
     gcloud compute backend-services describe "$service" \
       --project=<PROJECT_ID> --global \
       --format='table(name,securityPolicy,logConfig.enable)'
   done
   ```

   Every backend must reference `edge-block-policy` and report logging enabled.

4. **Check the reconciliation signal.** Inspect the latest
   `waf-config-reconcile` execution. Any `drift_detected` result is stronger
   evidence than log absence alone; follow `security-waf-config-drift.md`.

5. **Restore through Terraform.** If attachment or logging drifted, run the
   production provision workflow from the known-good main revision. Do not
   repair the policy manually unless Terraform cannot run during an incident.

## How to confirm it is resolved

- All three edge backends reference `edge-block-policy` with logging enabled.
- The reconciliation function returns `status: ok`.
- Fresh request logs again contain `enforcedSecurityPolicy.name`.
- The absence incident auto-closes after metric data resumes.

## When to escalate or ask for help

- Escalate immediately if a backend has no security policy attached.
- Escalate if logging is disabled or Terraform would delete unrelated
  resources while restoring the expected configuration.
- If configuration is correct but logs remain absent while traffic exists,
  treat it as a Cloud Logging/Monitoring incident and preserve raw evidence.
- Post in `ops-alerts`, tag `@platform-lead`, and bring the alert window,
  backend attachment output, and latest reconciliation result.

## Related ACs, signal sources, last-reviewed date

- Decision: ADR-110 WAF-integrity monitoring, issue #2729.
- Signal source: `labi_security/waf_cloud_armor_requests` log-based metric.
- Complementary signal: `security.waf_config_drift`.
- Last reviewed: 2026-08-08.
- Next review due: 2026-10-26.
