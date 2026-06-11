# Referrals API — task breakdown

**Source TDD:** `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-referral-api.md`
**Date of breakdown:** 2026-06-10
**Task count:** 5

## Tasks (in dependency order)

1. **`01-walking-skeleton-issuance.md`** — Stand up the `referrals-api` service end-to-end with the `POST /codes` issuance path behind the `referrals_api_enabled` flag, including schema for both tables and JWT auth wiring. *Depends on:* none.
2. **`02-idempotent-redemption.md`** — Implement the load-bearing `POST /codes/{code}/redeem` path with idempotency, transactional credit-ledger integration, and §7 failure-mode handling, including NFR load verification. *Depends on:* 01.
3. **`03-support-lookup-endpoint.md`** — Implement `GET /codes/{code}` for the support dashboard, gated on the ops JWT `support` claim. *Depends on:* 01.
4. **`04-observability-and-runbook.md`** — Service-wide metrics, log convention enforcement, alert deploy, and runbook entries per §9. *Depends on:* 02, 03.
5. **`05-staged-rollout.md`** — §8 rollout: staff-only enable → 1% → 10% → 100% US iOS ramp → separate web flip, with rollback drill. *Depends on:* 02, 03, 04.

## Dependency graph

```
01 (walking skeleton + schema + issuance + auth + flag)
 ├── 02 (redemption + credit-ledger + NFRs)
 │    ├── 04 (observability) ── 05 (rollout)
 │    └── 05 (rollout)
 └── 03 (support lookup)
      ├── 04 (observability) ── 05 (rollout)
      └── 05 (rollout)
```

Three engineers can pick up 02, 03, and observability scaffolding (04 prep) in parallel after 01 ships. 04 lands once 02 and 03 emit real telemetry. 05 is the final gate.

## TDD sections that could not be turned into tasks (open questions block them)

### §10.1 — Fraud / self-referral / multi-account abuse policy

> "Product owes us a decision on whether a referee on the same device, same payment instrument, or same household as the referrer counts as a valid redemption. **Until decided, we cannot scope the abuse-check task.**"

The TDD itself states this work cannot be scoped yet. No abuse-check task exists in this breakdown. Re-run the breakdown once product signs off on the policy and the TDD is updated.

### §10.2 — Code expiry behavior at the boundary (partial block)

> "When a code expires *while* a redemption is in flight (request received pre-expiry, processed post-expiry), do we honor it? Recommend: honor if the request arrived before `expires_at`; need product sign-off."

This does **not** block the redemption task entirely — the unambiguous case (request received strictly after `expires_at`) is implemented in task 02. The boundary case is explicitly carved out of task 02's scope and listed as a sub-bullet in its "Composes". Once product signs off on the recommendation, the boundary check is a small follow-up that can ride with task 02 if it hasn't shipped yet, or a small follow-up ticket if it has.

## What to look at first

- **Task 02 is the risk-front-loading task.** It opens a Postgres transaction across an external HTTP call to `credit-ledger-api`. That's the most failure-prone seam in the design. The NFR load test (3× peak = 600 redemptions/sec) belongs to this task specifically because that's where the load shows up; ledger latency under load is the most likely thing to break the design's assumptions.
- **The §10.2 carve-out in task 02 needs product sign-off before task 02 can be considered fully complete.** Flag this in standup with product.
- **Task 04 (observability) is cross-cutting by intent.** The qualifying test is documented inside the task: alert deploys span the whole service, runbook entries share on-call context, and the log-shape convention has to be enforced uniformly. If your team's preference is to fold telemetry into each feature task instead, the alert-config and runbook bullets still need a home — they don't ride cleanly with any single feature.
