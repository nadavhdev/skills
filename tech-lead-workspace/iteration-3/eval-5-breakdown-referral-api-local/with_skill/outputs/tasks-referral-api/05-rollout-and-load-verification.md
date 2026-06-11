### Flagged rollout with load verification at 3x peak

**One-liner:** Drive the §8 rollout from staff-only bake through 1% → 10% → 100% of US iOS traffic, with a load test at 3x peak before the 10% step to verify the §6 throughput and latency SLOs hold under promo-driven traffic.

**Composes:**
- Run staff-only with `referrals_api_enabled` on for staff JWTs for one week, per §8; collect the §9 metrics and confirm no alerts fire during the bake.
- Execute a load test at 3x peak (600 redemptions/sec, since §6 peak is 200/sec) against the staging deployment with `credit-ledger-api` staging as a dependency, and confirm the §6 numbers: p99 redemption < 250ms, p99 issuance/read < 150ms, and saturation < 70% at 4 pods (the §6 deploy shape) before stepping past 1%.
- Ramp US iOS traffic 1% → 10% → 100% over 5 days per §8, gated on the §9 alerts staying quiet at each step; the web client ramps behind a separate flag flip after iOS hits 100%, per §8.
- Confirm the §8 rollback contract holds: flipping `referrals_api_enabled` off mid-ramp stops new requests reaching the handler while in-flight synchronous redemptions complete (no orphaned redemption rows, no half-credited users).
- Coordinate with the credit-ledger team before the 10% step so their capacity headroom is confirmed for the traffic increase the ramp will create.

**TDD sections addressed:** §6 NFR latency and throughput (verified under load), §8 Rollout & migration (in full), §7.4 Promo-driven traffic spike (saturation alert verified during ramp).

**Depends on:** Issuance + redemption walking skeleton behind launch flag, Harden credit-ledger client for outage and slow-call failure modes, Support read endpoint for code lookup with ops-role auth, Observability, alerting, and runbook entries for the referrals surface

**Acceptance criteria:**
- Staff-only bake runs for one full calendar week with the flag on for the staff JWT cohort only and no §9 alerts firing.
- A load test at 600 redemptions/sec sustained for 10 minutes against the 4-pod staging deployment produces p99 redemption < 250ms, p99 issuance/read < 150ms, and saturation < 70%, with a written report archived in the team's runbook directory.
- The 1% → 10% → 100% iOS ramp completes over 5 days with each step gated on the prior step's metrics being green; any §9 alert firing during the ramp pauses progression and triggers the matching runbook entry.
- A mid-ramp flag flip-off (executed as a drill on staging during the 10% step) verifies the §8 rollback contract: no new redemptions accepted, all in-flight redemptions complete cleanly, no orphan rows in `referral_redemptions`.
- Web client ramp behind its own flag is executed only after iOS reaches 100% and is held there for at least 24 hours with no alerting.
- Credit-ledger team has signed off on capacity for the 10% and 100% steps before each is taken.
