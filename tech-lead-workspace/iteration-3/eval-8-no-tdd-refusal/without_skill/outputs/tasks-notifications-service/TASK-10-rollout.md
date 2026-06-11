# TASK-10: Rollout — feature flags, staged release, runbook

## Outcome
Both use cases go live safely with the ability to dial traffic up/down and a documented response plan when things go wrong.

## Scope
- Feature flags:
  - `notifications.push.order_shipped.enabled` (percentage rollout by user_id hash).
  - `notifications.email.promo.enabled` (kill switch).
- Staged rollout plan: 1% -> 10% -> 50% -> 100% with bake time and pass/fail criteria at each step.
- Runbook covering:
  - How to disable each channel fast (flag + provider-level pause).
  - How to drain / purge the queue.
  - How to requeue from DLQ.
  - Provider status page links and support contacts.
  - PagerDuty rotation & escalation.
- Load test against staging at expected peak (e.g. Black Friday shipment burst).
- Sign-off from: Product, Legal/Compliance, Security, SRE.

## Acceptance criteria
- [ ] Flags exist and have been exercised in staging.
- [ ] Runbook reviewed by on-call rotation.
- [ ] Load test report attached to ticket.
- [ ] Go/no-go checklist signed off by named owners.

## Dependencies
- TASK-05, TASK-06, TASK-08, TASK-09.

## Out of scope
- Marketing's launch plan and creative.

## Estimate
~3 days
