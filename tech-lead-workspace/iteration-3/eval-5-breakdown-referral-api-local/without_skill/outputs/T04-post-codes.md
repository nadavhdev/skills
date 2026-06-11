# T04 — `POST /codes` endpoint

**Estimate:** 2 days
**Depends on:** T02, T03 (and T08 for auth wiring, but can develop in parallel with T08 using a stub auth)
**TDD ref:** §4.1, §4.2, §4.3

## Outcome

Authenticated users can call `POST /codes` and receive a freshly issued,
unique, active referral code. Re-issuing for a user who already has an
active code invalidates the previous one and returns a new one.

## Scope

- Handler reads `issuer_user_id` from the validated JWT claims.
- Within one transaction:
  - Mark any existing `active` code for this user as `revoked` (or
    equivalent terminal status agreed in T02).
  - Generate a new code via T03 with a `check_exists` that queries
    `referral_codes.code`.
  - Insert the new row with `status='active'`, `created_at=now()`,
    `expires_at=now() + 90 days`.
- Respond `201 Created` with `{code, expires_at}`.
- On `CodeCollisionError` from T03: respond `503 Service Unavailable`
  with a generic body; this is rare-by-design and worth surfacing loud.
- On partial-unique-index violation (race: two concurrent issuances for
  the same user): retry the transaction once; if it fails again, 500.

## Out of scope

- Rate limiting per user (not in TDD; flag for follow-up if product
  raises it).
- Returning the share-link URL (client constructs it).

## Acceptance criteria

- [ ] Integration test: authenticated user issues a code, row exists with
      correct fields, response body matches.
- [ ] Integration test: same user calls `POST /codes` twice; the first
      code is no longer `active`, the second is `active`, only one
      active row exists for that user.
- [ ] Integration test: invalid / missing JWT → 401 (relies on T08).
- [ ] Unit test: collision retry exhaustion path returns 503.
- [ ] p99 latency in staging single-pod load < 150ms (validated more
      seriously in T10).
