### Support lookup endpoint with role-scoped auth

**One-liner:** Implement `GET /codes/{code}` for the support dashboard, gated on the `support` claim of an ops JWT, returning the code's issuer, status, expiry, and redemption record if any.

**Composes:**
- `GET /codes/{code}` returns issuer user id, status, `created_at`, `expires_at`, and the redemption record (referee, `credited_at`, idempotency key) if one exists.
- Requires an ops-issued JWT carrying the `support` claim per §4.5; user-scoped JWTs are rejected even if otherwise valid.
- Lookup of an unknown code returns 404; logs include `code` but never the requesting agent's JWT per §9.
- Read-only path: no write to either table, no call to `credit-ledger-api`.

**TDD sections addressed:** §4.1 (GET /codes/{code}), §4.5 AuthN/Z (support claim), §6 NFRs (read latency), §9 (log shape).

**Depends on:** Walking skeleton: issuance endpoint behind launch flag

**Acceptance criteria:**
- An ops JWT with `support` claim retrieves the full code record including redemption details when redeemed, and `null`/absent redemption block when not yet redeemed.
- A user-scoped JWT (no `support` claim) is rejected with the gateway's standard forbidden response.
- An ops JWT without `support` claim is also rejected.
- Lookup of a non-existent code returns 404 without leaking whether the code was ever issued vs. expired.
- p99 lookup latency under 150ms at expected support read volume.
- Structured logs include `code` and the requesting support actor id; logs do not include the JWT.
