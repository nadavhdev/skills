# T02 — Postgres schema + migration for referral tables

**Estimate:** 1 day
**Depends on:** T01
**TDD ref:** §4.2

## Outcome

`referral_codes` and `referral_redemptions` exist on the `growth` Postgres
cluster, with the constraints required by the design, deployed via the
standard migration tooling.

## Scope

- Migration creating:
  - `referral_codes(code PK, issuer_user_id, status, created_at, expires_at)`
  - Partial unique index on `referral_codes(issuer_user_id) WHERE status='active'`
  - `referral_redemptions(id PK, code FK → referral_codes.code, referee_user_id, idempotency_key, status, credited_at)`
  - Unique constraint on `referral_redemptions(code)`
  - Unique constraint on `referral_redemptions(idempotency_key, code)`
- Forward migration only; document rollback (drop tables) but do not auto-run.
- Apply in staging via the existing migration pipeline.
- Flip `GET /readyz` in `referrals-api` to verify a connection to the DB.

## Out of scope

- Any data backfill (there is none — new tables).
- Seed data for tests (handled per-endpoint in T04/T06/T07).

## Acceptance criteria

- [ ] Migration runs cleanly in staging.
- [ ] Inserting two active codes for the same `issuer_user_id` fails on the
      partial unique index.
- [ ] Inserting two redemptions for the same `code` fails on the unique
      constraint.
- [ ] Inserting two redemptions with the same `(idempotency_key, code)`
      fails on the idempotency unique constraint.
- [ ] `GET /readyz` returns 200 once DB is reachable.

## Notes

- Status enums (`active`, `redeemed`, `expired`, `revoked` for codes;
  `pending`, `credited`, `failed` for redemptions) — confirm exact values
  with the engineer picking up T04/T06 before writing the migration, to
  avoid a follow-up rename migration.
