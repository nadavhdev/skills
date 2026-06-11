### Idempotent redemption with credit-ledger integration

**One-liner:** Implement `POST /codes/{code}/redeem` as the load-bearing path: idempotent on `Idempotency-Key`, transactional against the redemption row, and dual-crediting both parties via `credit-ledger-api` with concurrent-redemption and ledger-outage failure modes handled per §7.

**Composes:**
- `POST /codes/{code}/redeem` accepts the referee's user id from their JWT and requires the `Idempotency-Key` header; missing or malformed key is rejected before any state change.
- Redemption opens a Postgres transaction, inserts into `referral_redemptions`, and calls `credit-ledger-api` with the same `Idempotency-Key` inside the transaction per §4.4. On ledger failure, the transaction rolls back and the redemption row is not persisted.
- Concurrent redemption of the same code is resolved by the `(code)` unique constraint: the losing request maps `unique_violation` to `409 Conflict` returning the existing redemption details per §7.
- Mobile double-tap retries with the same `Idempotency-Key` return the original result (no double-credit), enforced by the `(idempotency_key, code)` unique constraint per §4.2.
- Credit-ledger outage produces a loud failure to the client (no silent retry loop in our service) per §7; subsequent retries with the same `Idempotency-Key` are safe once the ledger recovers.
- Sub-bullet — blocked by §10.2 open question: behavior when a code expires *between* request receipt and processing. TDD recommends "honor if request arrived before `expires_at`" but flags it as needing product sign-off. Implementation of the boundary check must wait for that sign-off; current task implements the unambiguous case (request received after `expires_at` → reject).
- NFR verification for redemption: load test at 3× peak (i.e. 600 redemptions/sec) against the staged service, with the credit-ledger client running against a staged ledger instance.

**TDD sections addressed:** §3 High-level approach (load-bearing op), §4.1 (POST /codes/{code}/redeem), §4.2 (uniqueness constraints), §4.4 Credit posting, §6 NFRs (redemption latency, throughput), §7 Risks (credit-ledger outage, concurrent redemption, mobile double-tap, traffic spike).

**Depends on:** Walking skeleton: issuance endpoint behind launch flag

**Acceptance criteria:**
- Successful redemption returns 2xx, persists exactly one `referral_redemptions` row, and credits both referrer and referee via `credit-ledger-api`.
- Retrying with the same `Idempotency-Key` returns the original result and does not produce additional ledger calls or rows.
- Two concurrent redemptions of the same code result in one success and one `409 Conflict` with the existing redemption details in the body.
- A failure response from `credit-ledger-api` rolls back the transaction; no `referral_redemptions` row remains and no credit was posted.
- Redemption of an expired code (request received strictly after `expires_at`) returns the documented error. The boundary case (request received before, processed after) is **not** implemented in this task — it is gated on §10.2 product sign-off.
- p99 redemption latency under 250ms at 3× peak (600 redemptions/sec) in a load test against the staged stack, with ledger calls hitting the real staged ledger.
- Saturation telemetry exposed so the §9 alert "saturation > 70%" can fire (the alert config itself is delivered by the observability task).
- Structured logs at redemption include `code`, `idempotency_key`, `referee_user_id`; logs do not include the JWT or any other secrets per §9.
