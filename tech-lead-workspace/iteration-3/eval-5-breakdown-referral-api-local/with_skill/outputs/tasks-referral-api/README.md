# Referrals API — task breakdown

**Source TDD:** `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-referral-api.md`
**Date of breakdown:** 2026-06-10
**Task count:** 5 deliverable tasks (1 additional candidate task blocked by an open question — see below).

## Project conventions check

The working directory's `CLAUDE.md` describes conventions for the **csvkit**
CLI suite (one-tool-per-module subclassing `CSVKitUtility`, agate-based CSV
processing, `pytest` + `flake8` toolchain). The TDD being decomposed is for
a backend HTTP service (`referrals-api`), which is a different workload
entirely. None of the csvkit conventions apply. If this breakdown is being
executed inside a different repository whose conventions *do* govern API
work, surface that repo's `CLAUDE.md` / `CONTRIBUTING.md` / architecture
docs to the engineer picking up each task — those constraints should ride
on every task's "Composes" the way the TDD section citations do here.

## Tasks (dependency order)

| # | Title | File | Depends on |
|---|---|---|---|
| 1 | Issuance + redemption walking skeleton behind launch flag | `01-issuance-redemption-walking-skeleton.md` | none |
| 2 | Harden credit-ledger client for outage and slow-call failure modes | `02-credit-ledger-failure-mode-hardening.md` | 1 |
| 3 | Support read endpoint for code lookup with ops-role auth | `03-support-read-endpoint.md` | 1 |
| 4 | Observability, alerting, and runbook entries for the referrals surface | `04-observability-and-alerting.md` | 1, 2, 3 |
| 5 | Flagged rollout with load verification at 3x peak | `05-rollout-and-load-verification.md` | 1, 2, 3, 4 |

## One-line summary per task

1. **Walking skeleton** — schema, both write endpoints, idempotency, code generation, credit-ledger transactional call (happy path), 409-on-conflict, behind `referrals_api_enabled` default off. Ships an end-to-end demoable referral flow and retires the riskiest design seam (the transactional ledger call + idempotency) first.
2. **Credit-ledger hardening** — timeout, error classification, ambiguous-failure resolution via idempotency-key replay, no silent server-side retry. Makes the walking skeleton production-safe against the most likely incident source.
3. **Support read endpoint** — `GET /codes/{code}` for the support dashboard, gated on the ops `support` JWT claim. Independent fan-out from task 1's schema.
4. **Observability** — metrics, structured logs, three §9 alerts, three §9 runbook entries. Cross-cutting because §9 calls for a service-wide view of the referrals surface, not per-endpoint dashboards.
5. **Rollout** — staff bake → 3x-peak load test → 1%/10%/100% iOS ramp → web flag flip, gated on alerts staying quiet. Verifies the §6 NFRs under production-shaped load before full exposure.

## Dependency graph

```
1 (walking skeleton)
├── 2 (ledger hardening)
├── 3 (support read endpoint)
└── ──┬── 4 (observability) ── 5 (rollout)
   2 ─┤
   3 ─┘
```

Healthy shape: one foundation task (1), two independent fan-out tasks (2 and 3)
that can run in parallel by two engineers, one cross-cutting task (4) that
needs the surface area complete, then rollout (5) at the end.

## Open questions blocking work

The TDD has two open questions in §10. Their effect on this breakdown:

### §10.1 — Fraud / self-referral / multi-account abuse policy

> Product owes us a decision on whether a referee on the same device, same
> payment instrument, or same household as the referrer counts as a valid
> redemption. Until decided, we cannot scope the abuse-check task.

**Effect on this breakdown:** §2 already declares fraud/abuse out of scope
for the launch, so this open question does not block any of the five tasks
above. It defers a *future* "abuse check" task that is not part of this
breakdown. The five tasks here ship the launch surface; the abuse-check
task gets scoped (and broken down) after product resolves §10.1.

### §10.2 — Code expiry behavior at the boundary

> When a code expires *while* a redemption is in flight (request received
> pre-expiry, processed post-expiry), do we honor it? Recommend: honor if
> the request arrived before `expires_at`; need product sign-off.

**Effect on this breakdown:** **A sixth task — explicit expiry enforcement
at the request boundary — is blocked by this question and was not written.**
Task 1 currently treats `expires_at` as a stored field but does not enforce
expiry on the redemption path. Once product signs off on the
"honor-if-received-pre-expiry" recommendation (or a different rule), a
sixth task can be written to enforce that rule and add the matching
acceptance criteria to task 1. Until then, the redemption path will accept
expired codes — acceptable for staff bake and the 1% ramp, but **must be
resolved before the 10% step in task 5**.

## What to look at first

- **Task 1's idempotency + transactional ledger call.** This is the
  highest-risk seam in the design. If the assumption that "credit-ledger
  call inside the Postgres transaction with the same idempotency key" holds
  under realistic failure injection, the rest of the breakdown is
  straightforward. If it doesn't (e.g. ledger latency forces the transaction
  to hold locks too long), tasks 2 and 5 will need re-scoping and we should
  revisit the TDD §4.4 decision.
- **The §10.2 timing for task 5.** The 10% rollout step depends on product
  resolving §10.2 — flag this to product *now*, not when task 5 starts.
- **The cross-cutting observability task (4) depending on all three
  feature-shaped tasks.** This is deliberate per §9's service-wide framing,
  but it means task 4 can only start once tasks 1–3 are merged. If that
  serializes the team uncomfortably, an earlier "metrics for task 1 only"
  spike could be carved off — but my read is that per-feature metrics work
  would just get rewritten when 4 lands, so the current shape is correct.
