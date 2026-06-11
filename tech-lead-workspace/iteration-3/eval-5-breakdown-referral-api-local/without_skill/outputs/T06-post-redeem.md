# T06 — `POST /codes/{code}/redeem` endpoint with idempotency

**Estimate:** 3 days
**Depends on:** T02, T05 (T08 for auth, develop in parallel with stub)
**TDD ref:** §3, §4.1, §4.2, §4.4, §7

## Outcome

A referee can call `POST /codes/{code}/redeem` with an `Idempotency-Key`
header and a valid JWT. On success, both the referrer and the referee
receive a $10 credit; the call is safe to retry; concurrent redemptions
of the same code produce a clean `409 Conflict`.

## Scope

- Require `Idempotency-Key` header. Missing → `400`.
- Derive `referee_user_id` from JWT. Reject if referee equals issuer
  (`409` with a distinct error code — this is the only abuse rule we
  ship pre-policy; see TDD §10.1).
- Reject codes that are not `active` or whose `expires_at` is in the past
  with `410 Gone`. **Assumption** (TDD §10.2 recommended): a request
  received pre-`expires_at` is honored even if the credit posting lands
  after `expires_at`; we evaluate at row-read time.
- Within one Postgres transaction:
  1. `SELECT ... FOR UPDATE` the `referral_codes` row by code.
  2. Insert a `referral_redemptions` row with
     `(code, referee_user_id, idempotency_key, status='pending')`.
  3. Call `credit-ledger-api` via T05 **twice** — once for the issuer,
     once for the referee — each with a deterministic idempotency key
     derived from the request idempotency key (e.g.
     `{idempotency_key}:issuer`, `{idempotency_key}:referee`).
  4. Update the redemption row to `status='credited'`,
     `credited_at=now()`.
  5. Update the code row to `status='redeemed'`.
  6. Commit.
- Idempotency replay: if the unique constraint
  `(idempotency_key, code)` fires, **do not** treat as an error — load
  the existing redemption row and return the same response shape
  (`200 OK`) the original call would have produced.
- Concurrency: if the `referral_redemptions(code)` unique constraint
  fires for a *different* idempotency key, return `409 Conflict` with
  the existing redemption details.
- If any ledger call raises (T05 error types), roll back the transaction
  and return `502 Bad Gateway` (ledger unavailable) or `504` (timeout).
  The redemption row is gone, so the client retry with the same
  idempotency key is clean.

## Out of scope

- Fraud / abuse rules beyond self-referral (blocked by TDD §10.1).
- Metrics + alerting (T09 wraps this endpoint).

## Acceptance criteria

- [ ] Integration test: happy path — code goes `active → redeemed`,
      redemption row is `credited`, two ledger calls were made with the
      expected idempotency keys.
- [ ] Integration test: same `Idempotency-Key` replayed → same response,
      exactly one redemption row, exactly two ledger calls total across
      both requests (ledger client itself enforces idempotency on its
      side).
- [ ] Integration test: two concurrent redemptions of the same code with
      *different* idempotency keys → one wins, the other gets `409` with
      the winning redemption's details.
- [ ] Integration test: referee == issuer → `409` with self-referral
      error code, no ledger call made.
- [ ] Integration test: expired code → `410`, no ledger call made.
- [ ] Integration test: ledger 5xx → `502`, redemption row absent,
      retry with same idempotency key succeeds when ledger recovers.
- [ ] Integration test: missing `Idempotency-Key` header → `400`.
