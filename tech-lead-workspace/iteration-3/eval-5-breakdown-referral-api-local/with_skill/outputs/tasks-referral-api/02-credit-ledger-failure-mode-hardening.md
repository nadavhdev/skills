### Harden credit-ledger client for outage and slow-call failure modes

**One-liner:** Make the `credit-ledger-api` call path in redemption behave correctly when the ledger is slow, down, or returning ambiguous errors — so a ledger incident degrades to "cannot redeem" rather than corrupting referral state or silently double-crediting.

**Composes:**
- Bound the credit-ledger call with an explicit timeout aligned with the §6 redemption p99 SLO of 250ms (the ledger's own p99 SLO is 100ms per §6, so the budget is tight); on timeout, the redemption transaction rolls back per §4.4.
- Classify ledger responses into success, retryable failure, and non-retryable failure, and translate each into an explicit redemption outcome that the client retry policy (mobile resends with the same idempotency key, per §7.1) interacts with safely.
- Honor the §7.1 design call that there is no silent server-side retry loop on ledger failure — failures surface to the caller with a stable error shape, and the client decides whether to retry with the same `Idempotency-Key`.
- Ensure the same idempotency key is propagated to the ledger so that *its* retry semantics are safe on our resends, per §4.4.
- Cover the ambiguous-failure case (ledger call times out after the ledger has already accepted the credit): the next retry with the same idempotency key must converge on the original outcome because both our DB and the ledger key on it.

**TDD sections addressed:** §4.4 Credit posting, §6 NFR latency (redemption p99 < 250ms), §7.1 Credit-ledger outage, §9 Observability (credit-ledger latency metric, ledger-outage runbook).

**Depends on:** Issuance + redemption walking skeleton behind launch flag

**Acceptance criteria:**
- A simulated ledger outage (ledger returning 5xx for all calls) causes every in-flight redemption to roll back, leaving no `referral_redemptions` row in `status='credited'` and no partial credit on the ledger; the API returns a stable error shape the mobile client retry logic can act on.
- A simulated ledger slow-call (response time exceeding the configured timeout) is treated identically to an outage from the redemption's perspective — transaction rolls back, no partial state.
- A simulated ambiguous failure (ledger accepted the credit but our call timed out) is resolved on the client's retry with the same `Idempotency-Key`: the second attempt returns success and the user is credited exactly once on the ledger.
- No code path retries the ledger call server-side within a single redemption request — verified by inspecting the call graph and by a test that asserts exactly one ledger call attempt per redemption request.
- A redemption fails closed (returns an error, leaves DB and ledger consistent) under every failure-injection scenario in the runbook drill, with no orphan rows in `referral_redemptions` requiring manual cleanup.
- The `credit_ledger_latency_seconds` metric distinguishes timeout from non-timeout failures so the §9 alert "credit-ledger call p99 > 250ms for 10 min" can fire on the right signal.
