# T12 — Launch flag wiring + staged rollout plan

**Estimate:** 1 day
**Depends on:** T04, T06, T07, T09
**TDD ref:** §8

## Outcome

The service ships dark, gets turned on for staff, then ramps US iOS
traffic on a documented schedule, with web behind a separate flag.

## Scope

- Wire the flag client into the request path:
  - Flag `referrals_api_enabled` (default off) — gates the three
    endpoints at the gateway or handler layer (decide based on existing
    pattern).
  - Separate flag `referrals_api_web_enabled` — gates BFF traffic.
- Targeting rules:
  - Staff-only allowlist for week 1.
  - US iOS ramp: 1% → 10% → 100% over 5 days, each step gated by an
    on-call go/no-go with metrics from T09 as the criteria.
  - Web ramp on a separate flip, after iOS hits 100%.
- Rollback procedure documented: flip flag off, in-flight redemptions
  complete (synchronous), no data fix-up needed.
- Coordination:
  - Notify ledger team of expected redemption-call volume per step.
  - Notify support team that the dashboard read endpoint is live.

## Out of scope

- Marketing / comms (product owns).

## Acceptance criteria

- [ ] Both flags exist in the flag system and default off in production.
- [ ] Staff allowlist active and verified by at least 2 staff issuing +
      redeeming codes end-to-end in production.
- [ ] Ramp plan written up with explicit go/no-go metrics (error rate,
      ledger latency, saturation — same thresholds as T09 alerts) and
      the on-call rotation aware of it.
- [ ] Rollback drill: flag flipped off in staging, verified the three
      endpoints return the "feature disabled" response.
