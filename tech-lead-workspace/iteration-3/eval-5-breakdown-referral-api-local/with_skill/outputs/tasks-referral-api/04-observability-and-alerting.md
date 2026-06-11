### Observability, alerting, and runbook entries for the referrals surface

**One-liner:** Wire the metrics, structured logs, alerts, and runbook entries from §9 across all three endpoints so an on-call engineer can detect and diagnose a referrals incident from existing dashboards.

**Composes:**
- Emit the four metrics named in §9: `referral_issuance_total`, `referral_redemption_total{result}` (with `result` partitioned across success, idempotent replay, conflict, ledger-failure, and validation-failure outcomes), `credit_ledger_latency_seconds`, and the standard gateway request rate / error rate for the service.
- Emit structured JSON logs per §9 including `code`, `idempotency_key`, and `referee_user_id`, and never the JWT — verified by a log-redaction check.
- Wire the three alerts defined in §9: redemption error rate > 1% for 5 min; credit-ledger call p99 > 250ms for 10 min; saturation > 70% for 10 min.
- Author the three runbook entries listed in §9: ledger-outage drill (the procedure the on-call follows when task 2's ledger-down failure mode fires in production), idempotency-key collision diagnosis, and code-collision rate alarm investigation (tied to the §4.3 collision retry exhaustion error code surfaced by task 1).
- Cross-cutting by design: doing this per-feature would lose the holistic view of "what is the referrals surface doing right now", and the alerts in §9 are intentionally service-wide rather than per-endpoint.

**TDD sections addressed:** §9 Observability & operations (in full), §7.1 Credit-ledger outage (runbook), §7.2 Concurrent redemption (idempotency-key diagnosis), §4.3 Code generation (collision rate alarm).

**Depends on:** Issuance + redemption walking skeleton behind launch flag, Harden credit-ledger client for outage and slow-call failure modes, Support read endpoint for code lookup with ops-role auth

**Acceptance criteria:**
- All four §9 metrics emit on every request path they describe; `referral_redemption_total{result}` partitions cleanly across the five outcome classes above, with no requests landing in an unlabeled bucket.
- Structured logs for every endpoint contain `code` (when present), `idempotency_key` (on redeem), and `referee_user_id` (on redeem), and *never* the JWT or any token-derived secret — checked by an automated log-redaction test.
- All three §9 alerts are deployed in the alerting system with the named thresholds and durations, route to the existing growth on-call rotation, and have been tested with a forced-trigger drill.
- The three §9 runbook entries exist in the team's runbook, each one ends in a concrete action the on-call can take (not "investigate further"), and the ledger-outage runbook references the failure-mode behavior delivered by task 2.
- A dry-run alert simulation for each of the three alerts succeeds end-to-end (metric stub → alert fires → on-call paged in test channel).
