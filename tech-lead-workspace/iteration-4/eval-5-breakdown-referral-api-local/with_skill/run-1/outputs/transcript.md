Read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-referral-api.md`. Covers a new `referrals-api` synchronous service with three endpoints (issue / redeem / lookup), Postgres state on the growth cluster, idempotent redemption that calls the internal `credit-ledger-api`, and a staged flag-driven rollout to US iOS. 2 open questions noted.

Wrote the breakdown to `./tasks-referrals-api/` (5 task files plus a README index). I went with 5 tasks rather than padding to a higher number — folding code generation, the schema migration, and JWT auth wiring into the walking-skeleton task kept the foundation tax low, and the NFR load test rides with redemption because that's where the load shows up.

## Tasks (dependency order)

1. **Walking skeleton: issuance endpoint behind launch flag** — service scaffold, both Postgres tables, `POST /codes` with code generation, `referrals_api_enabled` flag, JWT auth. Ships a usable issuance path on its own. *Depends on: none.*
2. **Idempotent redemption with credit-ledger integration** — the load-bearing path. Idempotency, transactional ledger call, concurrent-redemption 409, double-tap retry safety, ledger-outage handling, NFR load verification at 3× peak. *Depends on: 01.*
3. **Support lookup endpoint with role-scoped auth** — `GET /codes/{code}` gated on the ops JWT `support` claim. *Depends on: 01.*
4. **Service-wide observability, alerts, and runbook** — §9 metrics, log convention enforcement, three alerts, three runbook entries. Cross-cutting by design; the README has the qualifying-test justification. *Depends on: 02, 03.*
5. **Staged rollout: internal QA → US iOS flag ramp** — staff enable for a week, then 1% → 10% → 100% over 5 days, separate web flip, rollback drill. *Depends on: 02, 03, 04.*

## Dependency shape

```
01 ── ┬── 02 ──┬── 04 ── 05
      │       │
      └── 03 ──┴── 04 ── 05
```

Healthy fan-out after the walking skeleton — 02 and 03 can be picked up by different engineers in parallel.

## Blocked by open questions

- **§10.1 — fraud/self-referral/multi-account abuse policy.** The TDD itself states this cannot be scoped until product decides. **No abuse-check task exists in this breakdown.** Re-run the breakdown once the policy lands.
- **§10.2 — code expiry at the boundary** (request received pre-expiry, processed post-expiry). Partial block: the unambiguous expiry case is implemented in task 02, but the boundary case is explicitly carved out and listed as a sub-bullet in task 02's "Composes". Needs product sign-off on the TDD's recommended behavior.

## What I'd look at first

- **Task 02 is the risk-front-loading task.** §4.4 has the service hold a Postgres transaction open across an external HTTP call to the credit-ledger — that's the most failure-prone seam in the design and the place where load assumptions are most likely to break. The 3× peak load test belongs there for exactly that reason; I'd prioritize verifying the seam before going wider.
- The §10.2 carve-out in task 02 needs product sign-off before task 02 is fully complete — worth flagging in standup with product so the answer arrives by the time the rest of redemption is in review.
- Task 04 is intentionally cross-cutting. If your team prefers folding telemetry into each feature task, the alert-config and runbook bullets still need a home — they don't ride cleanly with any single endpoint.
