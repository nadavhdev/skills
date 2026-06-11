# TASK-07: User notification preferences + unsubscribe

## Outcome
Users can control which channels and categories of notifications they receive, and every send path respects those preferences.

## Scope
- `user_notification_preferences` table: user_id, channel (`push` | `email`), category (`transactional` | `promotional`), enabled (bool), updated_at.
- Defaults: transactional ON, promotional ON (pending Legal confirmation per region).
- API:
  - `GET /v1/users/me/preferences`
  - `PATCH /v1/users/me/preferences`
- Public unsubscribe endpoint reachable from email link (signed token, no login required): `GET /unsubscribe?token=...`.
- Helper used by TASK-05 / TASK-06: `is_allowed(user_id, channel, category) -> bool`.
- Audit log of preference changes (who, when, from where).

## Acceptance criteria
- [ ] Disabling promo email blocks delivery in the next campaign launch.
- [ ] Disabling order push skips push in TASK-05 flow.
- [ ] Unsubscribe link from a real email flips the preference without requiring login.
- [ ] Preference changes appear in audit log.

## Dependencies
- TASK-02. Blocks: TASK-05, TASK-06.

## Out of scope
- Per-campaign opt-out granularity beyond category.
- Quiet hours / frequency caps (future ticket).

## Estimate
~3-4 days
