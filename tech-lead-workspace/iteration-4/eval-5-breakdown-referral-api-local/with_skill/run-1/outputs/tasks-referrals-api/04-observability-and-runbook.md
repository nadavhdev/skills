### Service-wide observability, alerts, and runbook

**One-liner:** Deliver the §9 observability surface — metrics, log shape, alerts, and runbook entries — as a single cross-cutting deploy that spans all three endpoints and the credit-ledger integration.

**Composes:**
- Metrics emitted across the service per §9: `referral_issuance_total`, `referral_redemption_total{result}`, `credit_ledger_latency_seconds`, gateway request rate and error rate. These names are exact per the TDD.
- Structured JSON log convention enforced service-wide: include `code`, `idempotency_key`, `referee_user_id` where applicable; never include the JWT or any other secret. Endpoint tasks emit the data; this task enforces the shape across all of them and lands the shared logging convention.
- Alert configuration deployed as a single change covering all three §9 alerts: redemption error rate > 1% for 5 min; credit-ledger p99 > 250ms for 10 min; saturation > 70% for 10 min. These are service-wide alarms that no single endpoint task owns.
- Runbook entries authored for all three §9 scenarios — ledger-outage drill, idempotency-key collision diagnosis, code-collision-rate alarm investigation — written as one coherent operations document because they share context and on-call procedures.
- Per the qualifying test for cross-cutting tasks: each bullet above either spans multiple endpoints (log convention, alert deploy, runbook) or owns a service-wide concern no single endpoint can own (saturation alert, error-rate alert across endpoints). None of these bullets would have ridden cleanly with a single feature task.

**TDD sections addressed:** §9 Observability & operations (all subsections), §7 Risks (informs runbook content).

**Depends on:** Idempotent redemption with credit-ledger integration, Support lookup endpoint with role-scoped auth

**Acceptance criteria:**
- All four named metrics are emitted from staging and visible in the metrics backend with non-zero values when traffic flows.
- All three alerts are deployed, paged through a manual fire-drill (e.g. inject failures, watch the alert fire in staging), and route to the on-call rotation.
- Log samples from staging show the required fields populated for issuance, redemption, and lookup paths, and contain no JWT substring (verified by a regex check on a log dump).
- Runbook is reviewable in the team's runbook system and an on-call engineer who hasn't seen the design can resolve a simulated ledger-outage page using only the runbook.
- A test that emits a fake JWT in a log line is caught by the log-shape convention (fails in CI or in a log-lint check).
