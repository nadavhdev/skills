# Cross-cutting NFRs — menu, not a checklist

This is a reference of NFRs to consider for a backend TDD. Use it as a menu:
include the ones that materially matter for this workload at this scale in
this org context. Omit irrelevant ones — but call out the omission briefly so
a reviewer knows you considered and dismissed them, not forgot them.

For each NFR you include, the TDD must state:

1. **Target** — a number, an SLO, or a concrete statement. "p99 < 200ms",
   "99.9% monthly", "PII encrypted at rest with KMS keys".
2. **Mechanism** — how the design meets the target.
3. **Verification** — how you'll know it's met (load test, alert, audit).

A line that says "highly scalable, robust, observable" is **not** an NFR; it's
a slogan. Replace it.

## Scalability

- **Capacity target.** Expected sustained RPS / throughput / data volume / row
  count. Peak vs steady-state. Growth horizon (6 months, 2 years).
- **Scaling axis.** Vertical (bigger box) vs horizontal (more boxes). For
  horizontal, what is the unit of scale and what prevents linear scaling
  (shared lock, hot key, stateful component, downstream limit).
- **Stateful bottleneck.** Where does state live, what's its scaling story
  (read replicas, sharding key, partition strategy)?
- **Back-pressure.** What happens when the system hits its limit — does it
  shed load, queue, fail fast, or fall over silently?

## Reliability & availability

- **Availability SLO.** 99.5? 99.9? 99.99? Per minute / per hour / monthly?
  Each "nine" costs roughly 10× more — be honest about what's needed.
- **Single points of failure.** Name every one. For each, the mitigation
  (replica, retry, fallback, graceful degradation).
- **Dependency failure handling.** For each external dependency: timeout,
  retry policy (with jitter), circuit breaker, fallback behavior. "Retries"
  alone without a budget is a footgun.
- **Idempotency.** Where retries can cause double-effects, what makes them
  safe — idempotency key, dedup table, conditional writes, natural idempotency.

## Resilience

- **Graceful degradation.** What partial functionality is preserved when a
  dependency is down? Stale cache? Read-only mode? Reject new but serve in-flight?
- **Bulkheads.** Are noisy-neighbor / one-tenant-takes-down-everyone scenarios
  isolated (per-tenant quotas, separate worker pools, separate connection pools)?
- **Timeouts everywhere.** Every external call has a timeout. Every internal
  blocking primitive has a timeout. No unbounded waits.
- **Backoff & jitter.** All retries use exponential backoff with jitter to
  prevent thundering herd on recovery.

## Performance

- **Latency targets.** p50, p95, p99 — call them out. Avoid "fast";
  number it.
- **Resource budget.** Memory, CPU, connections per instance. If memory
  is bounded by a design choice (streaming vs whole-file load), state it.
- **N+1 / fan-out hazards.** Where loops issue per-iteration calls, name
  them and the bound.

## Observability

- **Metrics.** Specific RED (rate, errors, duration) or USE (utilization,
  saturation, errors) signals with dimensions (route, tenant, dependency).
- **Logs.** Structured, with correlation/trace IDs. PII handling rules.
- **Traces.** Where distributed tracing is propagated; sampling rate.
- **Alerts.** SLO-based or symptom-based, not cause-based. What pages a
  human at 3am vs what is a ticket.
- **Runbook.** What's needed to operate this (deploy, roll back, common
  incidents). Link or placeholder.

## Security

- **AuthN.** Who calls this? With what credential (JWT, mTLS, API key,
  IAM role)? How is it validated?
- **AuthZ.** What can each caller do? Where is the policy enforced (gateway,
  per-handler, row-level)?
- **Secrets.** Where stored (vault, KMS), how rotated, who has access.
- **Data classification.** Does this touch PII, PHI, payment data, secrets?
  If yes: encryption at rest, encryption in transit, audit logging,
  retention, deletion / right-to-be-forgotten.
- **Input validation.** At every boundary (HTTP, queue, file). Don't trust
  upstream just because it's "internal".
- **Threat model surface.** Public internet? Internal-only? Third-party
  webhooks? Each implies a different threat model.
- **Supply-chain.** Dependency provenance, image signing, lockfiles for
  reproducible builds.

## Cost

- **Estimated unit cost.** Per request, per record, per tenant. Even a rough
  number forces a conversation.
- **Cost drivers.** Where is the biggest spend (compute, egress, storage,
  managed-service fees)? What knob controls it?
- **Cost ceiling / budget alert.** Threshold above which someone is paged.

## Compliance & privacy

- **Data residency.** Region requirements (EU data in EU, etc.).
- **Retention.** How long is data kept, what triggers deletion, audit trail.
- **Regulatory.** GDPR, HIPAA, SOC2, PCI — which apply, what they constrain.
- **Audit log.** What actions are recorded, by whom, for how long.

## Deployment & operations

- **Deploy strategy.** Blue/green, rolling, canary, feature-flag gated.
- **Rollback.** What does undo look like? Schema changes are usually the
  hard ones — call them out separately.
- **Feature flags.** Which flags gate this; default value; rollout plan.
- **Config.** Where it lives, how it changes, who can change it, audit trail.
- **Migrations.** Forward-only? Reversible? Zero-downtime story?

## Disaster recovery

- **RTO / RPO.** Recovery Time Objective, Recovery Point Objective. Real
  numbers, not "as fast as possible".
- **Backups.** What's backed up, where, how often, how is restore tested.
- **Multi-region story.** Active-active? Active-passive? None (and why
  that's OK)?

## Maintainability

- **Test strategy.** Unit, integration, contract, load, chaos — what's
  written and what's CI-gated.
- **Documentation.** Where the design lives, where API docs live, where
  the runbook lives.
- **Ownership.** Which team owns this in 12 months when the original
  authors are gone?

## Extensibility

- **Variation points.** Where the design *expects* future change — new
  message types, new tenants, new connector types — and how they plug in
  without a rewrite.
- **Versioning.** API, message schema, on-disk format. Backwards-compatible
  changes vs breaking changes, and how the latter is communicated.

## When to omit

- A 200 QPS internal admin API does not need a multi-region DR plan.
- A CLI utility does not have an availability SLO.
- A library does not have observability of its own (its host does).
- A one-off migration script does not need extensibility.

If you skip a section because it doesn't apply, **say so explicitly** in the
NFR section — a single line like "DR: not applicable; internal-only tool
with no uptime SLO". This signals deliberate omission vs forgetfulness.
