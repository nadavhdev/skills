# TASK-05: Order-shipped event -> push notification

## Outcome
When the orders service emits an `order.shipped` event, the user receives a push notification on every active device within ~1 minute.

## Scope
- Consumer subscribed to `order.shipped` topic/queue (confirm event bus — open question).
- Event contract documented and versioned (schema in shared registry if we have one).
- For each event: look up user's active device tokens, render push template (`order_shipped`), call `PushSender` for each token.
- Persist a `notification` row per send with status.
- Respect user preferences from TASK-07 (skip if user has disabled order push).
- Idempotency: same `event_id` processed twice must not produce duplicate notifications.

## Acceptance criteria
- [ ] End-to-end test: publishing a synthetic `order.shipped` event results in a push delivered to a test device and a `notification` row recorded.
- [ ] Replaying the same event does not produce a second send.
- [ ] User with no active devices is logged but does not error.
- [ ] User with push disabled in preferences is skipped, with a recorded reason.

## Dependencies
- TASK-02, TASK-03, TASK-07. Coordinates with orders team on event schema.

## Out of scope
- Email fallback when push fails (not in current product scope).
- Retry/DLQ mechanics (TASK-08).

## Estimate
~4 days
