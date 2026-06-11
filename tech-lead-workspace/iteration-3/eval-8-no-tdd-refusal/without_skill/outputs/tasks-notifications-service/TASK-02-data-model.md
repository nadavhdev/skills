# TASK-02: Data model + persistence

## Outcome
The service owns a database with the tables needed to track notifications, device tokens, and (later) delivery state.

## Scope
- Provision database (Postgres assumed; confirm with infra).
- Schema migrations tool wired into CI (e.g. Alembic / Flyway / Prisma — match team standard).
- Tables (initial):
  - `notification` — id, user_id, channel (`push` | `email`), template_key, payload (jsonb), status, created_at, sent_at, error.
  - `device_token` — id, user_id, platform (`ios` | `android`), token, last_seen_at, revoked_at.
  - `notification_template` — key, channel, subject, body, locale, version.
- Repository / DAO layer with unit tests against a real DB (testcontainers or equivalent).

## Acceptance criteria
- [ ] Migrations run forward and backward cleanly in CI.
- [ ] CRUD on each table covered by tests.
- [ ] Indexes on `notification(user_id, created_at)` and `device_token(user_id)`.
- [ ] PII (email, tokens) flagged in schema docs; encryption-at-rest confirmed at DB level.

## Dependencies
- TASK-01. Blocks: TASK-03, TASK-04, TASK-07, TASK-08.

## Out of scope
- User preference table (TASK-07).
- Delivery tracking columns beyond status/error (TASK-08).

## Estimate
~3-4 days
