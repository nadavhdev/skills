Broke the referrals TDD down into 12 task files, written to this directory
along with a `README.md` index. No Jira — pure local markdown, each task is
one file the team can copy/paste or convert later.

## Index of tasks

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

Total ≈ 17 engineer-days. With 2–3 engineers in parallel this is roughly
1.5–2 weeks of wall time given the dependency graph (the T01/T02/T03 fanout
unblocks most of the work after day 2).

## Suggested first-week pickup

- Engineer A: T01 → T02 → T04
- Engineer B: T03 → T05 → T06
- Engineer C (or A once T04 done): T07, T08

T09–T12 land in week 2 once the endpoints exist.

## Two things I did NOT create tickets for, on purpose

1. **Fraud / abuse policy (TDD §10.1).** Product still owes a decision on
   self-referral / same-device / household rules. I dropped the *bare
   minimum* abuse check (referee != issuer) into T06 so we are not totally
   exposed at launch, but anything beyond that is a follow-up epic once
   product decides. Don't let this block T06.
2. **Expiry-at-boundary semantics (TDD §10.2).** I baked the TDD's
   recommended behavior ("honor if the request arrived before
   `expires_at`") into T06 as an explicit assumption. If product comes
   back and rejects that recommendation, it becomes a small follow-up task
   — not worth a placeholder ticket right now.

Both are flagged in the T06 file and the `README.md` so whoever picks up
T06 sees them.

## File locations

All files are in:
`/Users/nhoze/.claude/skills/tech-lead-workspace/iteration-3/breakdown-referral-api-local/without_skill/outputs/`

- `README.md` — index, dependency graph, parked items
- `T01-service-scaffold.md` through `T12-launch-flag-rollout.md`
- `transcript.md` — this message
