# Recipient Lifecycle

Operational checklists for on-call responder enrollment, offboarding,
and monthly hygiene. Covers AC-LC-1..4 per architecture §9 and
UX §8.1-8.3. Lives alongside the six day-1 runbooks but is NOT
alert-linked (no Cloud Monitoring policy points here).

**Audience**: Platform admin (subject: a responder on the
`labalyst-prod-oncall` Pushover Group).

> **About the "roster" referenced below**: this checklist mentions
> `recipient-roster.md` repeatedly. That file is the append-only
> record of enrolled responders and lives in the **private**
> `labalyst/labalyst` repo at
> `documentation/operations/runbooks/recipient-roster.md` — it is
> deliberately not published in this public repo because it lists
> per-responder fallback emails. Platform admins follow the steps
> below from inside the private repo when running enrollments,
> offboardings, and monthly hygiene checks.

## Enrollment

### Prerequisites
- [ ] Responder has a personal smartphone (iOS or Android — Pushover
      ships on both).
- [ ] Responder has a personal external-domain email address
      (Gmail, Outlook.com, FastMail, ProtonMail, or any
      non-`labalyst.ai` domain — architecture §6.1 requires the
      fallback domain to differ from the primary platform domain per
      AC-AoA-4).
- [ ] Responder has read the six day-1 runbooks and knows where to
      find them.

### Steps
1. [ ] Responder creates a Pushover account, installs the app.
2. [ ] Responder sets the per-user Emergency-priority DND override
       for the Labalyst Pushover application within the app
       (AC-P1-2).
3. [ ] Platform admin adds the responder's user-key to the Pushover
       Group `labalyst-prod-oncall`.
4. [ ] Platform admin adds the responder's fallback Gmail to the
       Terraform `email_channels.fallback` list; `terraform apply`.
5. [ ] Platform admin triggers the synthetic-p1-weekly path
       manually (or waits for the next scheduled fire).
6. [ ] Responder confirms receipt on Pushover AND Gmail fallback,
       and acks on Pushover.

### Verification (required)
- [ ] Synthetic P1 received on Pushover. Ack recorded.
- [ ] Synthetic P1 received on Gmail fallback within 2 min of the
      Pushover delivery.
- [ ] Responder's user-key reference appears in `recipient-roster.md`
      with today's enrollment date.

### Evidence recording
- [ ] Add a row to `recipient-roster.md` (user_id, Secret Manager
      path reference to the user-key, fallback_email,
      enrollment_date).

### Rollback (if verification fails)
- [ ] Remove the user-key from the Pushover Group.
- [ ] Remove the fallback Gmail from Terraform
      `email_channels.fallback`; `terraform apply`.
- [ ] Document the failure reason in `recipient-roster.md` and retry
      after the fix is in.

### Failure-mode awareness
- Stale user-keys are an incident-data-leakage risk (PRD R-14) —
  do NOT add a responder whose verification has not completed.
- If the synthetic-p1-weekly path is broken at enrollment time, do
  NOT hand-wave the verification. Fix the path first (see
  `alerting-pipeline-broken.md`), then re-run the verification.

## Offboarding

### Prerequisites
- [ ] Team member departure has been announced, OR
- [ ] A device has been reported lost / stolen / retired.

### Trigger
- Team departure notification, OR
- Device-loss report, OR
- Compliance event requiring key rotation.

### Steps
1. [ ] Remove user-key from Pushover Group `labalyst-prod-oncall`.
2. [ ] Remove fallback Gmail from Terraform
       `email_channels.fallback`; `terraform apply`.
3. [ ] For a lost device: remote-sign-out the Pushover account via
       the Pushover web console; if credentials are compromised,
       rotate the user-key.
4. [ ] Expire any Pushover subscription keys tied to the departed
       account.
5. [ ] Update `recipient-roster.md` with the offboarding date and
       reason.

### Verification (required)
- [ ] Trigger synthetic-p1-weekly manually.
- [ ] Confirm the offboarded responder does NOT receive the
      synthetic on any device or email.
- [ ] Confirm remaining responders DO receive it.
- [ ] Record outcome in `recipient-roster.md`.

### Evidence recording
- [ ] Write today's date and PASS/FAIL to the `recipient-roster.md`
      offboarding column for this responder.

### Failure-mode awareness
- Stale user-keys are an incident-data-leakage risk (PRD R-14).
- A single missed offboarding step can let an ex-teammate receive
  `tenant_id` UUIDs indefinitely. Check the verification step, not
  just the removal steps.

## Monthly hygiene

**Cadence**: 1st of each month. **Assigned to**: Platform admin
(calendar-recurring). **Evidence**: An entry in
`recipient-roster.md` AND a written value at
`custom.googleapis.com/labi_alerting/hygiene_check_passed`.

### Prerequisites
- [ ] Current `recipient-roster.md` exists and matches the active
      Pushover Group + Terraform fallback list.
- [ ] On-call responders are reachable today (not all out).

### Steps (six, per architecture §9 and UX §8.3)
1. [ ] Open the current `recipient-roster.md`.
2. [ ] Trigger synthetic-p1-weekly manually.
3. [ ] For each responder listed:
   - [ ] Confirm Pushover delivery (app notification received on
         their device).
   - [ ] Confirm Gmail fallback delivery.
   - [ ] Confirm Pushover ack from their device.
4. [ ] Verify the fallback-only synthetic fired successfully within
       the last 4 weeks (check
       `custom.googleapis.com/labi_alerting/fallback_synthetic_ok`).
5. [ ] Run an HTTP-HEAD check on all six day-1 runbook URLs (both
       GitHub primary and GCS fallback paths). Expected: all HTTP
       200.
6. [ ] For each vendor credential in use (Pushover app token, Google
       Chat incoming-webhook URL, healthchecks.io ping URL): confirm
       it is still valid (each vendor exposes a "test" or "echo"
       endpoint).

### Verification (required)
- [ ] All six steps above completed and confirmed by the platform
      admin.
- [ ] Any FAIL has a follow-up issue open with the
      `platform-alerting-hygiene` label.

### Evidence recording
- [ ] Write today's date and PASS/FAIL to the `recipient-roster.md`
      monthly-check table.
- [ ] Write `1` to `labi_alerting/hygiene_check_passed` (or `0` if
      any step failed).
- [ ] If FAIL: open an incident-tracking issue labelled
      `platform-alerting-hygiene` and treat failure-to-confirm as a
      delivery defect — not an availability gap.

### Failure-mode awareness
- The hygiene check is the only regularly-scheduled exercise of the
  fallback-email path — if it is skipped, the fallback rots silently
  (codex Medium finding).
- An unack'd synthetic on any responder device surfaces both a
  delivery-path defect and a human-availability signal. Resolve
  both, separately.

## Related ACs, signal sources, last-reviewed date

- PRD AC references: AC-LC-1, AC-LC-2, AC-LC-3, AC-LC-4, AC-P1-2,
  AC-AoA-4, SM-11.
- Related docs: architecture §9, UX §8.1-8.3.
- Related scripts: `scripts/alerts/mark-fp.sh` (responder-side FP
  marker, main repo).
- Last reviewed: 2026-04-22
- Next review due: 2026-07-22 (quarterly cadence, platform lead).
