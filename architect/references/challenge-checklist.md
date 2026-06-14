# Challenge checklist — what to probe the design for

A thinking aid for the PRD/TDD grounding step. PRDs describe *what* and *why*;
they are usually thin on the engineering-relevant facts that drive risk. A good
TDD surfaces and answers those; a weak one inherits the PRD's silence. Walk this
list mentally and, for each item, ask two questions: **did the PRD specify it,
and did the TDD address it?** A gap in either is a candidate finding.

**Do not paste this list anywhere.** Pick the items that matter for this
workload; an internal CLI and a public streaming consumer share almost no
failure modes.

## 1. Problem & scope
- Is the design solving the actual PRD outcome, or a proxy for it?
- Is there a smaller design that still moves the PRD's metric? (→ simplification)
- Has the design silently grown or shrunk the scope the PRD set?

## 2. Users, load, access
- Who calls this, from where (public / internal / service-to-service)?
- Day-1 vs 6-month vs 2-year volume — peak and steady-state. Does the design
  hold across that range, or only day 1?
- Auth model — identified user, service token, anonymous? Is it in the design?

## 3. Data
- What does the design read / write / own? Single writer, or contention?
- Data classification — PII, payment, health, secrets? Handled accordingly?
- Volume, growth, retention. Consistency requirement (strong / read-your-writes
  / eventual)? Does the chosen store match it?
- Source of truth — if the design caches or derives data, what happens on
  divergence?

## 4. Performance & latency
- Latency budget for the caller. Downstream-call budget (DB, external API).
- Degradation mode under load — slow, error, drop, queue? Is it designed, or
  emergent?

## 5. Failure modes & blast radius
- Worst case if this is wrong / down / slow. Blast radius — one user, one
  tenant, the org, all customers?
- For each external dependency: if it's down for an hour, what does the user
  experience, and what does the data look like on recovery?

## 6. Idempotency & duplicates
- Can the same request/event arrive twice? (Almost always yes.) Does the design
  have an idempotency key and a defined behavior, or will duplicates corrupt
  state?

## 7. Ownership & operations
- Which team owns this in prod? Does the design depend on a system no one owns,
  or another team's unshipped work?
- Rollback story — especially for data changes that can't be un-done.
- Will on-call debug this from what the design exposes (logs/metrics/traces)?

## 8. Constraints & non-negotiables
- Cloud/region, compliance regime (GDPR/HIPAA/PCI/SOC2), budget ceiling, hard
  deadline. Does the design fit them, or quietly assume them away?

## 9. Integration points & contracts
- What systems does this talk to? Are their contracts documented, versioned,
  stable? Rate limits / quotas / SLAs respected?
- What new contract does this design expose, and who will depend on it?

## 10. Unstated assumptions
- What does the design assume about the existing system that may not be true?
- Where does a sentence like "this just…" hide a multi-month effort or a
  one-way-door decision?

## Triage
For each gap, classify: **blocks sign-off** (the design can't be built or will
fail the PRD without resolving it → `blocker`/`major` finding) · **should
resolve** (`minor`/`open-question`) · **not applicable** (skip silently). Be
ruthless about the first bucket — keep it to the few that genuinely change
whether this design is sound.
