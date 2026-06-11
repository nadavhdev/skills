# T11 — Runbooks

**Estimate:** 1 day
**Depends on:** T06, T09
**TDD ref:** §9

## Outcome

The on-call engineer has three runbook entries that map the TDD's named
failure modes to concrete diagnostic + remediation steps.

## Scope

Three runbook entries, in the standard runbook repo / format:

1. **Ledger outage drill** — symptoms (`502/504` redemption rate
   spike, `credit_ledger_latency_seconds` alarm), how to confirm it's
   the ledger and not us, who to page (ledger team), what we tell
   clients (retry with same `Idempotency-Key`), how to verify recovery.
   Include a tabletop drill script.
2. **Idempotency-key collision diagnosis** — what an idempotent replay
   looks like in logs vs an actual error, how to query a specific
   `idempotency_key` and read the redemption row, how to confirm only
   one credit was posted (cross-check ledger).
3. **Code-collision rate alarm investigation** — when the
   `referral_code_collision_retries_total` counter trends up, how to
   check whether code space is exhausted, what a sustained alert
   actually means (probably a generator bug, not exhaustion at our QPS).

## Out of scope

- Generic service-template runbook entries (deploy, rollback,
  scaling) — those come with the template.

## Acceptance criteria

- [ ] All three runbook entries linked from the service catalog page.
- [ ] Ledger-outage drill performed once with the on-call team; gaps
      fed back into the runbook.
- [ ] Idempotency runbook validated by walking through a real staging
      redemption replay.
