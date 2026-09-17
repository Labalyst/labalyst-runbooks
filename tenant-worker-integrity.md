# Runbook: Tenant-worker integrity incident

> **Alert class(es)**: `tenant_worker.oom`, `tenant_worker.child_instability`
> **Tier(s)**: P1
> **Last reviewed**: 2026-09-10 by {platform-lead}

## What this alert means

Cloud Monitoring found an out-of-memory kill, child-process SIGKILL, or worker
startup timeout in the exact environment's tenant worker pool. One underlying
interruption can emit several matching lines. The incident log window, not the
notification count, is authoritative.

An OOM is asynchronous. Treat every synchronous database connection held by an
affected task as suspect, even when the next error names another task. Do not
include `admin-worker` in the investigation: it has a separate workload,
credential set, and recovery path.

## First-response checks (in order)

1. **Preserve the incident window.** Record the alert's UTC start/end,
   environment, project, region, worker-pool revision, and instance ID before
   changing the deployment.

2. **Retrieve the tenant-worker interruption logs.** Set the three shell
   variables from the alert, then run:

   ```bash
   gcloud logging read \
     'resource.type="cloud_run_worker_pool" AND resource.labels.worker_pool_name="'"${ENVIRONMENT}"'-tenant-worker" AND (textPayload:"Out-of-memory event detected in container" OR textPayload:"SIGKILL" OR textPayload:"Timed out waiting for UP message")' \
     --project="${PROJECT}" --freshness=2h --format=json
   ```

   Group entries by instance ID, revision, and timestamp. Count one underlying
   interruption separately from its child-replacement messages.

3. **Correlate task lifecycle telemetry.** Query the same window for
   `jsonPayload.event="tenant_worker.task_memory"`. Match task ID, worker epoch,
   attempt, and instance ID. Record each lifecycle start without a terminal
   event and each acknowledged-late task that might be redelivered.

4. **Check collateral database and receipt state.** Search the affected
   instance and following task window for `pg8000`, protocol/message-state
   errors, closed sockets, unexpected transactions, and failed projection or
   embedding work. Inspect each interrupted operation's receipt and run record.
   A timed-out receipt must become terminal or enter its documented repair path.

5. **Contain the producer, then recover the revision.** Stop or rate-limit the
   burst-producing source. Let acknowledged-late tasks finish if memory is
   stable. If replacement continues, drain through the normal deployment and
   Terraform process and deploy a known-good revision. Do not mutate worker
   counts ad hoc; the autoscaler owns live instance count.

6. **Reconcile before retrying.** Verify that a retry cannot duplicate a
   completed side effect and that the original task ID and attempt history stay
   explainable. Run the affected projection or embedding health check on a
   clean pooled connection.

## How to confirm it is resolved

- The replacement revision is ready and its task lifecycle records close.
- Interrupted receipts have a truthful terminal state or documented repair.
- A clean pooled connection serves the affected health check.
- No new `pg8000` protocol errors, OOM, SIGKILL, or startup-timeout events
  appear during the observation window.

## When to escalate or ask for help

- Keep the producer limited and escalate to the platform lead if lifecycle
  telemetry is incomplete or measured memory headroom is unsafe.
- Escalate any unresolved receipt, duplicate-side-effect risk, or continuing
  database protocol error immediately in `ops-alerts`.
- Bring the UTC window, environment, revision and instance IDs, raw matching
  logs, lifecycle correlation, receipt states, and recovery actions.
- Capacity, concurrency, workload-isolation, and autoscaler changes require a
  separate evidence-backed decision; do not make one from this alert alone.

## Related ACs, signal sources, last-reviewed date

- Tracking: Labalyst issue #5430; deployed evidence follow-up #5432.
- Signal sources: `tenant_worker.oom`, `tenant_worker.child_instability`.
- Pool scope: `<environment>-tenant-worker` only.
- Last reviewed: 2026-09-10.
- Next review due: 2026-12-10.
