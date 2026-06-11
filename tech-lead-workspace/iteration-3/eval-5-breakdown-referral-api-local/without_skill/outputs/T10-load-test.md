# T10 — Load test at 3× peak for redemption + issuance

**Estimate:** 2 days
**Depends on:** T04, T06
**TDD ref:** §6

## Outcome

Documented evidence that the service meets its latency and throughput
SLOs at 3× peak before we open the rollout flag.

## Scope

- Test plan covering:
  - Issuance at 3× expected peak — confirm p99 < 150ms.
  - Redemption at 3× peak (TDD: 200/sec peak → test at 600/sec) —
    confirm p99 < 250ms.
  - Mixed workload (90% redemption / 10% issuance) at peak.
  - One scenario with the `credit-ledger-api` mocked to its own p99 SLO
    (100ms) so we measure our overhead in isolation.
- Run in staging against a representative pod count (4 pods per TDD §6).
- Record results in a short report stored next to the runbooks.

## Out of scope

- Soak / endurance testing (not in TDD; flag if reliability team asks).
- Chaos testing (ledger outage drill is part of T11).

## Acceptance criteria

- [ ] Issuance p99 < 150ms at 3× peak.
- [ ] Redemption p99 < 250ms at 3× peak.
- [ ] No 5xx in the mixed-workload run other than injected failures.
- [ ] Saturation < 70% at peak (matches the alert threshold from T09).
- [ ] Results document committed to the service repo (or wiki).
- [ ] If any SLO fails: file remediation tasks and rerun before T12.
