### Issuance + redemption walking skeleton behind launch flag

**One-liner:** Ship the end-to-end happy path for a single referral — issue a code, redeem it, post both credits via the credit ledger — behind `referrals_api_enabled` so we can demo internally without exposing the surface.

**Composes:**
- Persist referral codes and redemptions in the `growth` Postgres cluster with the two-table model in §4.2, including the partial unique index that enforces one active code per user and the unique constraint on `referral_redemptions(code)` that makes concurrent redemption losers visible.
- Expose `POST /codes` and `POST /codes/{code}/redeem` behind the existing API gateway, both gated on the user-scoped JWT per §4.5; both default-off behind `referrals_api_enabled`.
- Generate codes per §4.3 — 8-char Crockford base32, in-process with a bounded collision retry — and treat the retry as a backstop, not a hot path.
- Make redemption idempotent on the client-supplied `Idempotency-Key` header per §3, so mobile retries on flaky networks never double-credit.
- Post the $10 credit to both parties via the existing `credit-ledger-api` inside the redemption transaction per §4.4, passing the same idempotency key downstream so retries are safe on the ledger side too.
- Map the concurrent-redemption `unique_violation` on `referral_redemptions(code)` to `409 Conflict` with the existing redemption's details, per §7.2.

**TDD sections addressed:** §3 High-level approach, §4.1 Endpoints (POST /codes, POST /codes/{code}/redeem), §4.2 Data model, §4.3 Code generation, §4.4 Credit posting, §4.5 AuthN/Z (user JWT path), §7.2 Concurrent redemption, §7.3 Mobile double-tap, §8 Rollout (flag default off).

**Depends on:** none

**Acceptance criteria:**
- With the flag on for a test user, a `POST /codes` returns `{code, expires_at}` where `expires_at` is 90 days out; issuing a second code for the same user invalidates the first (the partial unique index permits exactly one `status='active'` row per issuer).
- A `POST /codes/{code}/redeem` with a valid user JWT, valid code, and an `Idempotency-Key` header credits both the referrer and the referee on the Stripe ledger exactly once; replaying the same request with the same idempotency key returns the original result without a second ledger call.
- A second distinct redemption attempt against an already-redeemed code returns `409 Conflict` with the existing redemption's identifying details (no second insert, no second ledger call).
- Requests without `Idempotency-Key` on the redeem endpoint are rejected at the handler with a 4xx; a JWT missing or invalid at the gateway never reaches the handler.
- With the flag off, all three write endpoints return the existing "feature off" response from the gateway; no rows are written.
- Code-collision retry exhausting all 5 attempts surfaces as a 5xx with a distinct error code so the runbook entry in task 5 can be wired to it.
- p99 issuance latency under 150ms and p99 redemption latency under 250ms in a single-pod local benchmark against a staging `credit-ledger-api`; the production-scale verification rides on the rollout task.
