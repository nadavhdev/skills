# Architect review lenses + severity model

The persona ("who you are when this skill is active") and the mandatory
conventions check live in `SKILL.md` — read it first. This file is the
*what to look at* and *how hard to push* of the review.

## How to use the lenses

Walk **every lens deliberately**, in the order below, before you finalize
findings. For each lens, ask: *"applied to this PRD + TDD + codebase, what does
this lens flag?"* Emit findings as you go, at calibrated severity (model below).
If a lens is clean for this design, note it to yourself and move on — **do not
file a finding to show you checked**. Coverage discipline is what makes the
review systematic rather than opportunistic; severity calibration is what makes
it trusted.

The lenses are ordered by what an architect review is *most* about. The first
two — does it solve the problem, and does it fit reality — are the ones a
tech-lead authoring the TDD is least able to check on themselves, which is
exactly why a fresh architect review exists.

These ten lenses are **cross-type**. Before walking them, you should already
have loaded the **type-specific** review file for this workload (`SKILL.md` step
5 — e.g. `review-event-driven.md`). Apply the two together: the type file tells
you the workload's one-way doors, hot spots, over-engineering tells, and which
NFRs to demand vs. flag as over-reach; the lenses below are how you reason about
any finding, typed or not. A type hot-spot that the design misses becomes a
finding under whichever lens fits (a missing DLQ → `risk`; an unearned new broker
→ `simplification`/`codebase-fit`).

### 1. PRD coverage & fidelity — does this design solve the stated problem?

The most important lens. For every requirement and success metric in the PRD:
does the TDD's design actually deliver it?

- **Dropped requirements** — a PRD requirement no part of the design addresses.
- **Silent scope drift** — the design solves a *different* (often bigger)
  problem than the PRD asked for, or quietly narrows it.
- **Metric served?** — if the PRD names a metric to move, does the design plausibly
  move it, or does it optimize something adjacent?
- **Smallest version** — is there a smaller design that still satisfies the PRD?
  (Feeds the simplification lens.)

Severity floor: a PRD requirement the design cannot deliver is at least `major`,
usually `blocker`.

### 2. Codebase fit & reuse — is this grounded in what already exists?

The lens a fresh reviewer catches and the author usually can't. The design lives
in a real codebase with real systems, patterns, and conventions.

- **Reinvention** — the design builds net-new what the codebase already has
  (a queue, a cache, an auth layer, a retry/backoff util, a job scheduler).
  Reuse beats rebuild unless the TDD names why the existing thing won't do.
- **Convention conflict** — the design violates a documented convention
  (`CLAUDE.md`, ADRs) or the de-facto pattern of neighboring code.
- **Coexistence & migration** — if it changes or sits beside existing systems,
  is the migration / dual-write / cutover path real, or hand-waved?
- **Ownership boundaries** — does it reach into another module's/service's
  internals rather than going through its contract?

Severity floor: reinventing or conflicting with an existing system without
justification is at least `major`; violating a hard convention follows the
convention-strength rule below.

### 3. Simplification — is there a materially simpler design?

You actively hunt for this; it is a primary job, not a courtesy.

- **Accidental complexity** — moving parts the problem doesn't require.
- **Premature abstraction** — generalized framework where one concrete case
  exists; pluggability nobody asked for.
- **New infra/deps not earned** — a new datastore/broker/service where an
  existing one or a library would do. Every new operational surface has a cost.
- **The simpler shape** — can you name a concretely simpler design that still
  satisfies the PRD? If so, that's the finding (point at it; don't author it).

Severity: usually `major` when the complexity is load-bearing and avoidable,
`minor` when it's a smaller over-build. Anchor it to the PRD requirement that
*doesn't* need the complexity.

### 4. Cross-cutting concerns (NFRs) — calibrated, and *specified*, not sloganed

Two failures live here, and most reviews catch only the first: NFRs that are
**missing** for a workload that needs them, and NFRs that are **present but not
actually specified**. Use the per-type file (its "demand if absent / flag as
over-reach" calibration) for *which* NFRs this workload needs; use the rules
below for *whether each one is real*.

**Which NFRs (calibration).** Cover only the ones this workload actually has —
the type file names them. Maximalist NFR review (DR for a CLI) is itself a
miscalibration → `over-engineering`. A NFR that matters for this type but is
**silently absent** is a finding; an NFR the TDD *explicitly* dismisses ("DR:
N/A — internal tool, no uptime SLO") is correct and gets **no** finding. Reward
the explicit omission; flag the silent gap — the difference tells you whether
the author considered it or forgot it.

**Whether each NFR is real (rigor).** For every NFR the TDD *does* include, it
must state three things (the NFR-menu bar): **Target** (a number or concrete
statement — "p99 < 200ms", "99.9% monthly", "PII encrypted at rest with KMS"),
**Mechanism** (how the design meets it), and **Verification** (how you'll know
it's met — load test, alert, audit). An NFR written as a slogan — "highly
scalable, robust, observable" — with no Target/Mechanism/Verification is *not an
NFR*; flag it (severity by how load-bearing). The **Verification** leg is the
most-skipped: an NFR you can't test is a wish, not a commitment.

**Show the math (quantification).** Where the TDD quantifies, the numbers must
add up: a latency budget must **sum under its target** (stages totalling 240ms
against a 150ms target is a `performance` finding, usually `major`); a capacity/
throughput claim must be consistent with the stated scaling axis and bottleneck;
a cost claim needs a range, not hand-waving. A budget that doesn't sum is a
concrete, takeable, principal-level catch — exactly the kind that earns trust.

The concerns themselves (cover the subset the type file flags):
- **Security & data** — auth model, trust boundaries, data classification (PII /
  payment / health), secret handling, source-of-truth / consistency, retention.
  **Match depth to exposure:** a public/internet-facing surface (webhook, public
  API) demands a full threat model; a cross-tenant internal API demands authz +
  tenant isolation; a local CLI with no secrets needs little. A real
  trust-boundary gap is a `blocker` regardless of surface — but don't demand a
  full threat model where there is no boundary.
- **Reliability & resilience** — dependency-down behavior, blast radius,
  idempotency on writes, retry/backoff, graceful degradation.
- **Observability** — can on-call debug this from what the design exposes?
  RED/USE signals on the load-bearing paths; alerts that page on symptoms.
- **Scalability** — holds at the stated *and* foreseeable load? Name the concrete
  limit (hot partition, lock contention, unbounded memory in an aggregation).
- **Cost** — for cost-driven workloads (batch, scheduled-pull, async-worker), is
  the unit cost (per record / poll / worker) named and bounded, and the dominant
  cost driver identified? Unnamed unit cost on a volume-scaled design is a `cost`
  finding.
- **Testability** — does the design name a test strategy fit for the type
  (golden datasets for batch, contract tests for an API, load/chaos where an NFR
  needs proving)? This is the Verification leg made concrete; its absence is a
  `testability` finding.

Severity: a missing trust-boundary/security consideration is `blocker`. Slogan /
missing-verification / over-reach are `major`/`minor` by impact. Tag with the
**specific** area (`security`, `data`, `scalability`, `performance`,
`observability`, `cost`, `testability`) — not a generic one.

### 5. Evolvability & future impact — one-way vs two-way doors

You see past this feature. Spend scrutiny on the expensive-to-reverse decisions.

- **One-way doors** — schema shapes, public/cross-team contracts, data-ownership
  choices, wire formats. These are costly to change after launch; they deserve
  the hardest look. A reversible decision that's slightly suboptimal is fine.
- **Foreseeable next step** — the obvious follow-on feature or the 10x scale: does
  this design extend to it, or force a rewrite? Don't demand the design *build*
  for a future that may not come — demand it not *preclude* it cheaply.
- **Versioning & compatibility** — for anything other consumers depend on, is
  there a versioning / backward-compat story?

Severity: a one-way door taken without naming the tradeoff is at least `major`.

### 6. Risk & failure-mode coverage

The risks the design under-addresses.

- **SPOFs and data loss** — single points of failure; scenarios where data is
  lost or corrupted; missing idempotency where duplicates are inevitable.
- **Dependency on the unowned** — the design relies on a system no team owns, or
  on another team's unshipped roadmap. Org-level risk.
- **Recovery** — when the bad thing happens, can state be recovered (replay from
  broker retention, re-run a batch, restore from backup)? (Deployment/migration
  *safety* itself is lens 7.)

Severity: an unaddressed data-loss or correctness risk is `blocker`/`major`.

### 7. Operational readiness — deployment, rollback, migration

How this design reaches production and backs out safely. The author, heads-down
on the happy path, under-covers this more than any other lens — which is exactly
why a fresh reviewer should walk it deliberately.

- **Migration safety** — are schema/data changes designed for online /
  zero-downtime, or do they lock a table / block writes under load? A migration
  that blocks writes on a live table is the classic prod incident.
- **Rollback** — what does undo look like, *especially* for data changes that
  can't be un-done? A forward-only change on a risky path with no rollback story
  is a finding.
- **Rollout control** — canary / blue-green / feature-flag gating for anything
  risky or user-facing. An all-at-once cutover of a load-bearing change with no
  gate is a finding.
- **Graceful shutdown / drain** — for long-running workloads (workers, consumers,
  APIs), does a deploy drain in-flight work, or kill it and lean on redelivery
  (and is that redelivery idempotent)?

Severity: a migration that blocks writes under load, or a data change with no
rollback path, is `blocker`/`major`. Tag `operability`.

### 8. Tradeoff honesty & alternatives considered

A principal TDD names real alternatives and rejects them with reasons.

- **Alternatives present?** — did the design consider the obvious other shapes
  (different store, different pattern, buy-vs-build) and honestly reject them?
- **Tradeoffs honest?** — when the TDD says "we'll use X", does it name what X
  *costs* (operational surface, ordering constraints, the exactly-once myth), or
  sell it as free?
- **Hidden cost** — a cost the design glosses that will bite operationally.

Severity: usually `minor`/`major` — a missing-alternatives section is `minor`;
a chosen approach whose unstated cost is load-bearing is `major`.

### 9. Contracts & interfaces

The public and cross-team surface the design introduces or changes.

- **Defined & versioned** — are the contracts (API shapes, event schemas, CLI
  surface) specified well enough to build and to depend on?
- **Backward compatibility** — does a change break existing consumers in the
  codebase/org? Unannounced breaking changes to a shared contract are serious.
- **Blast radius** — how many consumers does a contract change ripple to?

Severity: an unannounced breaking change to a shared contract is `blocker`;
under-specified contracts are `major`/`minor` by how load-bearing they are.

### 10. Open questions & readiness

Can the team actually start building from this?

- **Load-bearing unknowns** — a question the TDD leaves open whose answer would
  change the design. These must be resolved (or explicitly de-risked) before
  sign-off — surface them as `open-question` findings.
- **Decisions punted** — the TDD lists a decision as "TBD" that the design
  depends on. Not the same as a genuinely deferrable detail.

Severity: a load-bearing open question is `major` (it blocks sign-off until
resolved); a deferrable detail is `minor` or not a finding at all.

## Severity model

Four severities. Calibration matters more than count. The loop converges on
sharp, anchored findings and spins on fuzzy ones.

| Severity | Meaning | Examples |
|---|---|---|
| `blocker` | The design as written will fail to meet the PRD, ship a correctness/security/data-loss risk, take an expensive one-way door blindly, or break a shared contract. **Must resolve before sign-off.** | PRD's core requirement undeliverable by this design; trust boundary with no auth; data-loss path with no mitigation; schema choice that can't support the stated query pattern. |
| `major` | Departs from the PRD/codebase/good design in a way that will cause real problems or rework but isn't fatal. **Must resolve or be explicitly accepted with reason.** | Reinvents an existing service without justification; missing idempotency where duplicates are inevitable; materially simpler design exists; NFR the workload needs is uncovered. |
| `minor` | A real issue worth fixing but not blocking. | Alternatives section missing; a smaller over-build; an observability gap on a non-critical path. |
| `nit` | Taste or polish. The author may defer. | Section naming; a diagram that would help; wording of a tradeoff. |

### Hard severity rules (apply every time, not by mood)

These keep the loop from re-arguing severity each round:

1. **A PRD requirement the design cannot deliver = at least `major`** (usually
   `blocker` if it's a core requirement).
2. **Any design-level security gap = `blocker`.** Missing auth/trust-boundary
   handling, secret-in-design, PII without classification. No "minor security."
3. **An unannounced breaking change to a shared/public contract = `blocker`**
   unless the PRD or TDD explicitly authorizes the break.
4. **A documented-convention violation tracks the convention's strength:**
   `MUST` → `blocker`, `SHOULD` → `major`, `MAY` → `minor`. If strength is
   implicit, lean conservative — it was documented for a reason.
5. **A data-loss or data-correctness risk with no mitigation = `blocker`.**
6. **A schema/data migration that blocks writes on a live table under load, or a
   risky data change with no rollback path = `blocker`/`major`** (`operability`).

### Calibration rules (judgment)

- If you wouldn't defend a `blocker` to another principal ("would they agree this
  *must* change before we build?"), demote it.
- Don't pad. Three anchored findings beat twelve speculative ones.
- "I'd design it differently" with no consequence named is a `nit` at most —
  usually it's not a finding. If it really matters, name the consequence.
- Concede freely. If the TDD's choice is defensible once you read the codebase
  and PRD carefully, there is no finding. Challenging only when rightful cuts
  both ways.

## Verdict

Binary — the loop needs an unambiguous signal:

- **`SIGN_OFF`** — no `blocker` or `major` findings remain. `minor`/`nit` may
  remain. SIGN_OFF means *"this design is sound, grounded, and ready to break
  down into implementation tasks."*
- **`REQUEST_CHANGES`** — at least one `blocker` or `major` finding stands. No
  "sign off with conditions" — a condition you care about is a `major` finding.

If you want to sign off "but they should really also…", you have a finding
you're softening. Mark it and request changes.
