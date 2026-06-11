# TASK-09: Observability — metrics, logs, alerts

## Outcome
On-call can answer "is the service healthy?", "are notifications going out?", and "what broke?" within minutes, without reading code.

## Scope
- Metrics (Prometheus / vendor — match team standard):
  - `notifications_sent_total{channel,category,status}`
  - `notifications_send_duration_seconds{channel}` (histogram)
  - `notifications_queue_depth{queue}`
  - `notifications_dlq_size`
  - `provider_errors_total{provider,error_class}`
- Structured logs: include `notification_id`, `user_id` (hashed), `channel`, `event_id` for correlation.
- Tracing: spans across consumer -> render -> provider call (OTel if standard).
- Dashboard: one per channel (push, email) + one campaign overview.
- Alerts (page on-call):
  - Send success rate < 95% over 10 min.
  - DLQ size growing > X over 15 min.
  - Provider 5xx rate > threshold.
  - Consumer lag > threshold.

## Acceptance criteria
- [ ] Dashboards exist and are linked from the service README.
- [ ] Synthetic failure in staging fires the expected alert.
- [ ] A notification can be traced end-to-end from event to provider response via its ID.

## Dependencies
- TASK-05, TASK-06 (need traffic to be meaningful, but can start in parallel).

## Out of scope
- Customer-facing analytics on campaign performance (open rate, CTR) — separate ticket if needed.

## Estimate
~3 days
