# T09 — Metrics, structured logs, and alert rules

**Estimate:** 2 days
**Depends on:** T04, T06, T07
**TDD ref:** §9

## Outcome

Operators can see exactly how the service is doing in the existing
metrics + logs stack, and get paged when the TDD's alert thresholds are
breached.

## Scope

- Metrics (Prometheus / OpenMetrics, matching org conventions):
  - `referral_issuance_total{result=success|collision_exhausted|error}`
  - `referral_redemption_total{result=success|conflict|expired|self_referral|ledger_unavailable|ledger_timeout|idempotent_replay|error}`
  - `credit_ledger_latency_seconds` (histogram)
  - `referral_code_collision_retries_total` (counter from T03 wrapper)
  - Standard HTTP request rate / latency / error rate (likely auto-emitted
    by framework, confirm).
- Structured logs:
  - One log line per request with `code`, `idempotency_key`,
    `referee_user_id`, `issuer_user_id`, `result`, `latency_ms`.
  - **Never** log the JWT.
- Alert rules (deployed to the existing alerting stack):
  - Redemption error rate > 1% for 5 min → page.
  - `credit_ledger_latency_seconds` p99 > 250ms for 10 min → page.
  - Pod saturation (CPU or in-flight requests) > 70% for 10 min → ticket.
  - Code-collision retries trending upward (e.g. > 10/min) → ticket.
- Dashboards: one Grafana dashboard with issuance volume, redemption
  volume by result, ledger latency, error rate, saturation.

## Out of scope

- Tracing wiring (assumed auto from the service template; verify and
  open a follow-up only if missing).

## Acceptance criteria

- [ ] All metrics above appear in staging Prometheus with non-zero
      values under synthetic load.
- [ ] A staged failure (ledger killed for 1 minute) fires the redemption
      error-rate alert in the alerting stack's test channel.
- [ ] Log audit on staging: zero lines contain a JWT, all redemption
      lines contain the documented fields.
- [ ] Dashboard PR merged.
