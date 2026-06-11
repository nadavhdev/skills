### Walking skeleton: issuance endpoint behind launch flag

**One-liner:** Stand up the `referrals-api` service end-to-end with the `POST /codes` issuance path behind the `referrals_api_enabled` flag, including schema for both tables and JWT auth wiring.

**Composes:**
- New `referrals-api` synchronous JSON service deployed behind the existing API gateway, gated by the `referrals_api_enabled` launch flag (default off).
- Postgres migration on the `growth` cluster creating `referral_codes` and `referral_redemptions` per §4.2, including the partial unique index `(issuer_user_id) WHERE status='active'` and the `referral_redemptions` uniqueness constraints. Schema lands ahead of redemption so the redemption task can build on it without a second migration.
- `POST /codes` issues an 8-character Crockford base32 code per §4.3 with a collision-retry loop bounded at 5 attempts; re-issuing for the same user invalidates the previous active code via the partial-unique index.
- User-scoped JWT validation wired at the gateway, role check at the handler per §4.5. This auth path is the foundation subsequent endpoints reuse.
- Returns `{code, expires_at}` with the 90-day TTL from §2.

**TDD sections addressed:** §3 High-level approach, §4.1 (POST /codes), §4.2 Data model, §4.3 Code generation, §4.5 AuthN/Z (issuance side), §8 Rollout (flag mechanism).

**Depends on:** none

**Acceptance criteria:**
- With the flag on, an authenticated user calling `POST /codes` receives a valid 8-character Crockford base32 code and an `expires_at` 90 days out.
- A second `POST /codes` for the same user invalidates the prior code (DB shows only one active row for that user) and the prior code is no longer redeemable in lookups.
- With the flag off, the endpoint returns the gateway's standard not-enabled response and writes nothing to the database.
- Requests without a valid user JWT are rejected at the gateway before reaching the handler.
- Code-generation collision-retry loop is exercised in a test that forces a collision and verifies bounded retries; exhausting 5 retries returns a 5xx with a logged collision-rate signal.
- Migration runs forward and backward cleanly in a staging deploy; rollback leaves the cluster in its prior state.
- p99 issuance latency under 150ms in a smoke test (NFR §6, issuance branch).
