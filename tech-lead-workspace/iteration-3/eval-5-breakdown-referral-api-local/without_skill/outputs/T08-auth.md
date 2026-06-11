# T08 — JWT validation + role-scoped auth

**Estimate:** 1 day
**Depends on:** T01
**TDD ref:** §4.5, §6 (Security)

## Outcome

`referrals-api` validates JWTs at the handler layer (gateway already
validates signature) and enforces role scoping so that `GET /codes/{code}`
requires the `support` claim and the other two endpoints require a normal
user-scoped JWT.

## Scope

- Middleware / dependency that:
  - Parses the JWT (already verified upstream by the gateway — we trust
    the signed header, but re-verify signature with the gateway's public
    key for defense in depth).
  - Extracts `user_id` and claim set onto the request context.
  - Rejects missing / malformed token → `401`.
- A `requires_role("support")` decorator / guard used by `GET /codes/{code}`.
- Make sure the JWT is **never** logged (TDD §9).

## Out of scope

- Token issuance (handled by the existing auth service).
- Refresh / revocation (handled upstream).

## Acceptance criteria

- [ ] Unit test: valid user JWT → handler runs with `user_id` populated.
- [ ] Unit test: missing token → `401`.
- [ ] Unit test: malformed / bad-signature token → `401`.
- [ ] Unit test: user JWT against `requires_role("support")` → `403`.
- [ ] Unit test: ops JWT with `support` claim against `requires_role("support")` → handler runs.
- [ ] Log scrubbing test: a request with a JWT does not produce any log
      line containing the token value.
