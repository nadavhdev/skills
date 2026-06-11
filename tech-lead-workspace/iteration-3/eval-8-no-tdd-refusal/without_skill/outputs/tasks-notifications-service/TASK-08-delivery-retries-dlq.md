# TASK-08: Delivery tracking, retries, and DLQ

## Outcome
Transient failures are retried with backoff; permanent failures land in a dead-letter queue with enough context to triage; every notification's lifecycle is queryable.

## Scope
- Add lifecycle states to `notification`: `pending`, `sending`, `sent`, `failed_retryable`, `failed_permanent`, `suppressed`.
- Classify provider errors: retryable (5xx, rate limit, timeout) vs permanent (invalid token, hard bounce, suppressed address).
- Retry policy: exponential backoff with jitter, max N attempts (e.g. 5), capped at 1h between attempts. Configurable per channel.
- DLQ (separate queue/table) for permanently failed sends; includes payload, last error, attempt count.
- Internal API: `GET /v1/notifications/{id}` to inspect a single notification's history.
- Internal API: `GET /v1/notifications?user_id=...&since=...` for support.

## Acceptance criteria
- [ ] Simulated provider 500 retries and eventually succeeds within policy.
- [ ] Simulated invalid token / hard bounce moves to `failed_permanent` without retry.
- [ ] DLQ items are visible in dashboard and can be requeued.
- [ ] Support can look up a user's recent notifications and see why one failed.

## Dependencies
- TASK-02, TASK-03, TASK-04. Used by TASK-05, TASK-06.

## Out of scope
- Automatic remediation of DLQ.
- Customer-facing notification history UI.

## Estimate
~4 days
