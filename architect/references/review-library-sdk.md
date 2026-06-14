# Architect review aspects — library / internal SDK

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-library-sdk.md` first — that is
the standard this TDD was (or should have been) written against. Use its required
sections, NFR applicability, and pitfalls as the baseline checklist. This file
adds the architect lens.

The defining fact of this workload: **the public API surface is the product.**
Almost every design decision here is a one-way door, because consumers compile/
link against it and you can't see all of them.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **The public API surface.** Every exported signature, type, and name is a
  contract. Breaking it ripples to every consumer. Is the surface minimal and
  deliberate, or does it leak internals that lock in future implementation?
- **Error / exception model.** Exceptions vs result types, the exception
  hierarchy — consumers write handling against it.
- **Semantic-versioning commitments.** What counts as breaking; the deprecation
  path. Sets the maintenance contract.
- **Dependency footprint.** Every dependency the library pulls in is imposed on
  every consumer. A heavy or conflicting dep is a `dependency` finding.

## Type-specific hot spots (review questions)
- **Backward-compatibility policy.** Is there an explicit versioning + deprecation
  story, or will the first change break consumers (`contract`)?
- **Minimal dependency footprint.** Is each runtime dependency justified, or could
  stdlib do it? Consumers inherit these (`dependency`).
- **Thread-safety contract.** Is it stated whether the API is safe to call
  concurrently? Silence here causes subtle consumer bugs.
- **No global mutable state.** Hidden singletons / module-level state make the
  library un-composable and test-hostile (`code-quality`/`maintainability`).
- **Extensibility without breaking.** Can consumers extend behavior without the
  library breaking them on the next release?

## Over-engineering & simplification tells
- An elaborate plugin / abstraction system for what has one concrete use today
  → point at the concrete shape (`over-engineering`).
- Configuration knobs for everything ("flexibility") that nobody asked for and
  that widen the support surface.
- Premature generality (interfaces with a single implementation) that locks in a
  shape before the second case exists.

## Calibration — NFRs
- **Flag as over-reach if present:** availability SLOs, disaster recovery,
  multi-region — a library has no runtime of its own. Their presence is a
  miscalibration.
- **Demand if absent:** backward-compatibility / versioning policy, dependency
  justification, thread-safety statement, a clear error model, a **per-call
  overhead + import-time-side-effect budget** (you run in the host's process),
  and a **test matrix** (host language/framework versions) including a
  no-import-side-effects test.
