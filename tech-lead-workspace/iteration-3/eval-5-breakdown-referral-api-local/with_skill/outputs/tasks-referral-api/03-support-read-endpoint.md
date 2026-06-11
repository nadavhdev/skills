### Support read endpoint for code lookup with ops-role auth

**One-liner:** Expose `GET /codes/{code}` for the support dashboard so an agent can look up a code's issuer, status, expiry, and any redemption record, gated to the ops `support` JWT claim.

**Composes:**
- Implement the read path defined in §4.1: return the code, issuer, status, expiry, and the redemption record if one exists.
- Enforce the §4.5 auth split: this endpoint requires the ops JWT with the `support` claim; user-scoped JWTs (used by `POST /codes` and `POST /codes/{code}/redeem`) must be rejected here.
- Default-off behind the same `referrals_api_enabled` flag as the write endpoints from task 1, so rollout of the read path can ride the §8 ramp without a separate flag.
- Hit the §6 issuance/read p99 < 150ms target; this is a single Postgres SELECT joined to at most one redemption row, so the budget is generous, but capture it so regressions are visible.
- Never log the JWT and never echo the JWT back in responses, per §9 logging conventions.

**TDD sections addressed:** §4.1 Endpoints (GET /codes/{code}), §4.5 AuthN/Z (support JWT path), §6 NFR latency (read p99 < 150ms), §9 Observability (structured logs without JWT).

**Depends on:** Issuance + redemption walking skeleton behind launch flag

**Acceptance criteria:**
- A request with a valid support JWT returns the full code record (code, issuer user id, status, expiry, redemption record if any) in a stable JSON shape; an unknown code returns 404.
- A request with a valid *user* JWT (no `support` claim) returns 403, regardless of whether that user owns the code.
- A request with no JWT or an invalid JWT is rejected at the gateway and never reaches the handler.
- Structured logs for the endpoint include `code` but never the JWT or any token-derived secret, per §9.
- p99 < 150ms for the read path in a single-pod local benchmark against staging data; production-scale verification rides on the rollout task.
- With the flag off, the endpoint returns the gateway's "feature off" response and performs no DB read.
