# T07 — `GET /codes/{code}` support-dashboard read endpoint

**Estimate:** 1 day
**Depends on:** T02 (T08 for the `support` role check)
**TDD ref:** §4.1, §4.5

## Outcome

Support staff can look up a code and see its full lifecycle: who issued
it, status, expiry, and the redemption record if any.

## Scope

- Auth: requires an ops JWT with the `support` claim. Without it → `403`.
- Single DB read joining `referral_codes` and `referral_redemptions`.
- Response shape:
  ```
  {
    "code": "...",
    "issuer_user_id": "...",
    "status": "active|redeemed|expired|revoked",
    "created_at": "...",
    "expires_at": "...",
    "redemption": null | {
      "referee_user_id": "...",
      "status": "...",
      "credited_at": "..."
    }
  }
  ```
- Unknown code → `404`.

## Out of scope

- Search by issuer or referee (not in TDD; flag if support asks).
- PII redaction (TDD says user ids are the only PII and are fine for
  support).

## Acceptance criteria

- [ ] Integration test: active code, no redemption → `redemption: null`.
- [ ] Integration test: redeemed code → `redemption` populated.
- [ ] Integration test: ops JWT without `support` claim → `403`.
- [ ] Integration test: user-scoped JWT (not ops) → `403`.
- [ ] Integration test: unknown code → `404`.
