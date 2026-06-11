# TASK-03: Push provider integration

## Outcome
Service can send a push notification to a known device token via the chosen provider (FCM / APNs / vendor) behind a clean internal interface.

## Scope
- Decide provider (open question — see README). Default assumption: FCM for Android, APNs for iOS, via a single SDK or vendor abstraction.
- Implement `PushSender` interface with `send(device_token, title, body, data) -> SendResult`.
- Credential management via secrets store (no keys in repo or env files).
- Device token registration endpoint: `POST /v1/devices` (auth required) — upserts into `device_token`.
- Device token revocation endpoint: `DELETE /v1/devices/{id}`.
- Handle provider-side invalid-token responses by marking `device_token.revoked_at`.

## Acceptance criteria
- [ ] Integration test against provider sandbox sends a real push to a test device.
- [ ] Unit tests cover error paths: invalid token, rate-limited, provider 5xx.
- [ ] Secrets pulled from vault / parameter store, not env files.
- [ ] Endpoint contracts documented (OpenAPI or equivalent).

## Dependencies
- TASK-01, TASK-02. Blocks: TASK-05.

## Out of scope
- Triggering on order-shipped events (TASK-05).
- Retry policy beyond a single send attempt (TASK-08).

## Estimate
~4-5 days
