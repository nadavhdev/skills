# Architect review aspects — API serving (synchronous)

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-api-serving.md` first — that is
the standard this TDD was (or should have been) written against. Use its
required sections, NFR applicability, and pitfalls as the baseline checklist.
This file adds the architect lens.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **The public API contract.** Endpoint shapes, request/response schemas, status
  codes, error format. Every consumer couples to these; changing them later is a
  breaking change with org-wide blast radius. Are they specified well enough to
  build *and* to depend on?
- **Auth model.** Who calls this (user, service, anonymous)? AuthN/Z chosen at
  the trust boundary is hard to retrofit. A design-level gap here is a `security`
  `blocker`.
- **Data model / schema.** The persisted shape and its query patterns. A schema
  that can't serve the PRD's access pattern is a `blocker`.
- **Sync-vs-async boundary & pagination.** What's returned inline vs via a job/
  callback; the pagination scheme. Both lock the contract.

## Type-specific hot spots (review questions)
- **Latency budget + downstream-call budget.** Is the caller's latency
  expectation stated, and does the sum of downstream calls (DB, external APIs)
  fit under it? Watch for N+1 and unbounded queries (`performance`).
- **Idempotency on writes.** Can the same request arrive twice? Is there an
  idempotency key and defined behavior, or will retries double-write (`data`)?
- **Error contract.** Are status codes and error shapes defined, consistent, and
  non-leaky (no stack traces / internal detail to clients)?
- **Rate limiting / overload behavior.** Degradation mode under load — shed,
  queue, 429? Designed or emergent?

## Over-engineering & simplification tells
- A microservice split where a module in the existing service would do.
- GraphQL / gRPC where plain REST satisfies the PRD.
- CQRS / event-sourcing introduced without a requirement that earns it.
- Multi-region active-active for a low-QPS internal API → point at the simpler
  single-region shape (`simplification`).
- A new datastore where an existing one in the codebase fits (`codebase-fit`).

## Calibration — NFRs
- **Demand if absent:** auth model, latency budget (**that sums under its
  target** — do the arithmetic), idempotency on writes, observability for the
  request path, an **availability SLO**, a **deployment/rollback story** (no
  write-blocking schema migration; zero-downtime cutover; canary), and **contract
  tests** for the public surface.
- **Flag as over-reach if present without a driving requirement:** multi-region
  active-active, five-nines availability, exactly-once across services — for a
  modest internal API these are unearned complexity.
