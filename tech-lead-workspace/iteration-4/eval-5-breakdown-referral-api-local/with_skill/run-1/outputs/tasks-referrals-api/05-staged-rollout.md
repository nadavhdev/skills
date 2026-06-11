### Staged rollout: internal QA → US iOS flag ramp

**One-liner:** Execute the §8 rollout — staff-only enable for one week, then 1% → 10% → 100% of US iOS traffic over 5 days, with web behind a separate flag flip — with rollback procedure verified.

**Composes:**
- Enable `referrals_api_enabled` for staff users only and hold for one week of internal QA per §8. Track usage and any incident in the runbook scratchpad.
- Mobile flag ramp on the US iOS cohort: 1% → 10% → 100% over 5 days. Each step holds long enough to observe the §9 alerts at the new traffic level before advancing.
- Web flip handled as a separate flag change *after* the iOS ramp completes per §8 ("Web behind separate flag flip").
- Rollback drill in staging *before* the production ramp: flip the flag off, confirm in-flight redemptions complete synchronously, confirm no new traffic reaches the service.
- Go/no-go criteria per ramp step: redemption error rate stays under 1%, credit-ledger p99 stays under 250ms, saturation stays under 70% — exactly the §9 alert thresholds.

**TDD sections addressed:** §8 Rollout & migration, §9 (alert thresholds drive go/no-go).

**Depends on:** Idempotent redemption with credit-ledger integration, Support lookup endpoint with role-scoped auth, Service-wide observability, alerts, and runbook

**Acceptance criteria:**
- Staff-only enable runs for a full week with no Sev-1/Sev-2 incidents attributed to the service.
- Rollback drill in staging shows: flag flip propagates within the documented window, in-flight redemptions complete, no new requests accepted.
- Each iOS ramp step (1%, 10%, 100%) is held long enough that the §9 alerts would have fired if breached, and the documented go/no-go thresholds were met before advancing.
- Web flip is a separate, documented change — not bundled with the iOS ramp.
- A timestamped rollout log is captured (start of each step, observed metrics at hold, decision to advance) and stored with the runbook.
