# Referrals API — task breakdown

Source TDD: `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-referral-api.md`

These tasks decompose the Referrals API TDD into deliverable units the team can
pick up next week. Each task is sized to be completable by one engineer in
1–3 days. Dependencies are noted at the top of each task.

## Recommended sequencing

```
T01 service scaffold ──┐
T02 schema migration ──┴─► T03 code generation ──► T04 POST /codes
                                                        │
                                                        ▼
                                       T05 ledger client ──► T06 POST redeem
                                                                    │
                                                                    ▼
                                                          T07 GET /codes/{code}
                                                                    │
       ┌────────────────────────────────────────────────────────────┤
       ▼                ▼                  ▼                ▼        ▼
T08 auth/JWT    T09 observability    T10 load test    T11 runbooks    T12 launch flag & rollout
```

## Blocked / parked

- **Fraud / abuse policy** (TDD §10.1) — blocked on product. No task created.
  Will become its own follow-up epic once policy is decided.
- **Expiry-at-boundary semantics** (TDD §10.2) — recommendation in TDD is
  "honor if request arrived before `expires_at`". Carried as an assumption in
  T06; flip to a real task only if product rejects the recommendation.

## Index

| ID  | Title                                                          | Est. | Depends on |
|-----|----------------------------------------------------------------|------|------------|
| T01 | Bootstrap `referrals-api` service scaffold                      | 1d   | —          |
| T02 | Postgres schema + migration for referral tables                 | 1d   | T01        |
| T03 | Code generator (8-char Crockford base32, retry-on-collision)    | 1d   | T01        |
| T04 | `POST /codes` endpoint                                          | 2d   | T02, T03   |
| T05 | `credit-ledger-api` client wrapper                              | 1d   | T01        |
| T06 | `POST /codes/{code}/redeem` endpoint with idempotency           | 3d   | T02, T05   |
| T07 | `GET /codes/{code}` support-dashboard read endpoint             | 1d   | T02        |
| T08 | JWT validation + role-scoped auth (`support` claim)             | 1d   | T01        |
| T09 | Metrics, structured logs, and alert rules                       | 2d   | T04, T06, T07 |
| T10 | Load test at 3× peak for redemption + issuance                  | 2d   | T04, T06   |
| T11 | Runbooks: ledger outage, idempotency collision, code collision  | 1d   | T06, T09   |
| T12 | Launch flag wiring + staged rollout plan                        | 1d   | T04, T06, T07, T09 |
