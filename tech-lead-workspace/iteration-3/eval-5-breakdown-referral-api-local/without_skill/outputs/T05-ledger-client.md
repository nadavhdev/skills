# T05 — `credit-ledger-api` client wrapper

**Estimate:** 1 day
**Depends on:** T01
**TDD ref:** §3, §4.4, §7

## Outcome

A small, well-tested client that calls `credit-ledger-api` to post a $10
credit, forwarding the caller's `Idempotency-Key`. It is the only place
in `referrals-api` that knows how to talk to the ledger.

## Scope

- One method: `post_credit(user_id, amount_cents, idempotency_key, reason) -> CreditResult`.
- HTTP client with:
  - Connection + read timeouts sized so total budget fits the redemption
    p99 (target ledger p99 100ms; cap total at 200ms).
  - **No retries inside this service.** Per TDD §7, redemptions fail
    loud and the *client* retries with the same idempotency key.
- Typed errors: `LedgerTimeoutError`, `LedgerUnavailableError` (5xx),
  `LedgerClientError` (4xx — surface body for log).
- Auth: service-to-service token from existing secret store.

## Out of scope

- Any caching.
- Circuit breaker (revisit if the load test shows we need it).

## Acceptance criteria

- [ ] Unit test against a mock HTTP server: success path returns
      `CreditResult` with ledger txn id.
- [ ] Unit test: ledger returns 5xx → `LedgerUnavailableError`.
- [ ] Unit test: ledger returns 4xx → `LedgerClientError`, body captured.
- [ ] Unit test: ledger does not respond within timeout →
      `LedgerTimeoutError`.
- [ ] Unit test: `Idempotency-Key` header is forwarded verbatim on every
      call.
- [ ] Unit test: no internal retry happens on any failure path.
