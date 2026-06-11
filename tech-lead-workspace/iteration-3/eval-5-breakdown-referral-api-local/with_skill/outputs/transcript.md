Read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-referral-api.md`. Covers the three-endpoint synchronous referrals service, the two-table Postgres model with idempotent redemption against the credit ledger, the §6 latency/throughput NFRs, and the §8 staff-bake → US iOS ramp rollout. 2 open questions noted (§10.1 fraud/abuse policy, §10.2 expiry-boundary semantics).

Wrote the breakdown to `./tasks-referral-api/` — 5 tasks + a README index, in dependency order:

1. **Issuance + redemption walking skeleton behind launch flag** — the foundation task. Schema, both write endpoints, idempotency, code generation, the credit-ledger transactional call (happy path), and 409-on-concurrent-redemption, all behind `referrals_api_enabled` default off. Folded into one task because doing schema-only or endpoint-only as a standalone task would produce no shippable value, and the riskiest design seam (transactional ledger call + idempotency-key replay) needs to be retired early.
2. **Credit-ledger failure-mode hardening** — timeout, error classification, ambiguous-failure resolution, no silent server-side retry. Carved out from task 1 because the failure-mode work is substantial enough on its own and §7.1 is the most likely incident origin once this ships.
3. **Support read endpoint** with ops-role JWT. Independent fan-out from task 1.
4. **Observability + alerting + runbooks** — cross-cutting, depends on 1/2/3. Justified per §9's service-wide framing.
5. **Rollout + 3x-peak load verification** — staff bake → load test → 1%/10%/100% iOS ramp → web flip. Verifies §6 under production-shaped load.

**1 task blocked by an open question and intentionally not written:**

§10.2 (code expiry behavior at the request boundary) blocks a sixth task that would enforce expiry on the redemption path. The breakdown does not invent an answer — until product signs off on the "honor-if-received-pre-expiry" recommendation in the TDD (or picks a different rule), task 1 will accept expired codes. That's tolerable for the staff bake and the 1% step of task 5, but **must be resolved before the 10% step**. Flag it to product now, not when task 5 starts.

§10.1 (fraud/abuse policy) does not block this breakdown — §2 already declared fraud/abuse out of scope for the launch. It defers a *future* abuse-check task that gets scoped after product resolves §10.1.

**Project conventions note:** the working-directory `CLAUDE.md` is for csvkit (CLI tools subclassing `CSVKitUtility`), which is a different workload entirely and does not apply to a backend HTTP service. The README in `tasks-referral-api/` calls this out. If you'll execute these tickets inside a different repo (the actual `referrals-api` codebase), surface *that* repo's conventions to whoever picks up each task.

**Look at first:**

- **Task 1's transactional ledger call + idempotency seam.** If the §4.4 "ledger call inside the Postgres transaction with the same idempotency key" assumption holds under realistic failure injection, the rest of the breakdown is straightforward. If it doesn't (e.g. ledger latency forces the transaction to hold locks too long under load), tasks 2 and 5 will need re-scoping and we should revisit the TDD §4.4 decision.
- **Task 4's dependency on 1/2/3.** The cross-cutting observability work can only start once the surface is in. That's correct per §9, but if it serializes the team uncomfortably, say so and we can revisit — though my read is that per-feature metrics work would just be rewritten when task 4 lands.
