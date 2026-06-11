# T01 — Bootstrap `referrals-api` service scaffold

**Estimate:** 1 day
**Depends on:** none
**TDD ref:** §3, §4

## Outcome

A new service `referrals-api` exists in the standard service template,
deploys to staging behind the API gateway, responds to `GET /healthz` with
200, and has CI green (lint, unit-test, build, container image).

## Scope

- Create the repo / service directory using the org's service template.
- Wire it to the existing API gateway with a stub route for `/referrals/*`.
- Health and readiness probes.
- CI: build, unit tests, container image, deploy to staging.
- Config plumbing for: Postgres DSN (placeholder), credit-ledger base URL
  (placeholder), launch-flag client.

## Out of scope

- Any business endpoints (handled in T04, T06, T07).
- DB schema (T02).
- Auth middleware (T08).

## Acceptance criteria

- [ ] Service builds and the container image is published by CI.
- [ ] `GET /healthz` returns 200 in staging through the gateway.
- [ ] `GET /readyz` returns 503 until DB connectivity is verified (so T02
      can flip it green).
- [ ] Service appears in the standard service catalog / on-call rotation
      placeholder.

## Notes

- Follow the existing service-template conventions exactly; this task is
  pure plumbing and should not introduce new patterns.
