# JSON contracts for the architect review loop

The loop is driven programmatically: you (the architect) emit a review as JSON,
the tech-lead responder emits its response as JSON, and you parse it to decide
the next round. Both contracts are below. Honor them exactly — a malformed
payload forces a re-spawn, loses context, and degrades the review.

**In Mode A (driven loop)** the review is consumed programmatically — **return
raw JSON only, no prose before or after, no markdown code fences.** **In Mode B
(review-only)** a human is reading: lead with a readable rendering of the review
(verdict, summary, findings grouped by severity with their anchors), and keep the
JSON as the machine record in the `review-rounds.md` ledger. Either way the JSON
is produced — it is the state carrier the ledger and any later targeted
re-review depend on.

## 1. The architect review (what you emit each round)

```json
{
  "verdict": "SIGN_OFF" | "REQUEST_CHANGES",
  "round": <int, 1-indexed>,
  "summary": "<2-4 sentences: what's strong about this design, and the headline concern(s)>",
  "findings": [
    {
      "id": "<round.index, e.g. '1.3'>",
      "severity": "blocker" | "major" | "minor" | "nit",
      "area": "<one of the closed vocabulary below>",
      "summary": "<one-line, scannable>",
      "detail": "<2-6 sentences: what's wrong, what to change, and why it matters>",
      "anchor": "<the PRD section/line, TDD section, codebase fact, or named failure mode this is grounded in — never null on a blocker/major>",
      "simpler_alternative": "<if area is simplification/over-engineering: the concretely simpler shape you're pointing at, else null>"
    }
  ],
  "prior_findings_review": [
    {
      "ref": "<prior finding id, e.g. '1.3'>",
      "status": "resolved" | "not_resolved" | "conceded" | "deadlocked",
      "note": "<one line: how the revision closed it, why it didn't, why you conceded, or the state of the disagreement>"
    }
  ]
}
```

### Field rules

- **`prior_findings_review` is omitted entirely on round 1** (there are no prior
  findings). On every round ≥ 2 it is **required** and must have one entry per
  finding still open from the previous round. Its statuses:
  - `resolved` — the tech-lead's revision genuinely closes the finding. Drop it.
  - `not_resolved` — the revision is absent, partial, or misses the root cause.
    The finding **must also reappear in `findings`** at its original severity.
  - `conceded` — the tech-lead's evidence-based challenge holds; you withdraw
    the finding. You challenge only when rightful, and that cuts both ways — a
    good rebuttal is a resolution, not a defeat. Do **not** re-list it in
    `findings`.
  - `deadlocked` — both sides hold with evidence across rounds. Keep it in
    `findings` at its severity; the loop will surface it to the user (see
    `review-loop.md`).
- **`anchor` is the most important field.** A finding anchored to "PRD §3.2
  requires 50K orders/day; TDD §5 single-writer Postgres design caps at ~X" is
  takeable. A finding anchored to taste gets pushed back on. Never leave it null
  on a `blocker` or `major`.
- **`simpler_alternative`** is where you *point at* the simpler design without
  authoring it — name the shape ("reuse the existing outbox table instead of a
  new Kafka topic"), not a full design.
- **`verdict`** is `SIGN_OFF` only when zero `blocker`/`major` findings remain
  across both new findings and any `not_resolved`/`deadlocked` priors.
- `id` uses `round.index` so findings are referenceable across rounds.

### Closed `area` vocabulary (pick the closest; do not invent)

| `area` | What goes here |
|---|---|
| `prd-coverage` | A PRD requirement dropped, mis-served, or scope-drifted |
| `codebase-fit` | Reinvents/conflicts with an existing system; violates a convention; coexistence/migration hand-waved |
| `simplification` | A materially simpler design satisfies the PRD |
| `over-engineering` | Complexity/abstraction/new infra not earned by the PRD |
| `cross-cutting` | An NFR/cross-cutting concern under-covered (use the specific tag below if one fits) |
| `security` | Design-level auth/trust-boundary/secret/PII gap |
| `data` | Data model, consistency, source-of-truth, retention, idempotency at design level |
| `scalability` | Won't hold at stated or foreseeable load; named limit |
| `performance` | Latency/throughput/resource gap; a budget that doesn't sum under its target; N+1 / fan-out |
| `observability` | On-call can't debug this from what the design exposes |
| `cost` | Unit cost (per request/record/poll/worker) unnamed or unbounded; a cost driver the design ignores |
| `evolvability` | One-way door taken blindly; precludes the obvious next step cheaply |
| `risk` | Under-addressed failure mode, SPOF, data-loss |
| `operability` | Deployment/rollback/migration safety — schema migration that blocks writes, no rollback for data, no canary/feature-flag, no graceful drain |
| `testability` | No test strategy appropriate to the workload; an NFR with no verification (how we'll know it's met) |
| `contract` | Public/cross-team interface under-specified or broken (versioning, backward-compat) |
| `ownership` | Depends on an unowned/other-team system; creates ambiguous ownership |
| `tradeoff` | Alternatives not considered, or stated tradeoffs dishonest/incomplete |
| `open-question` | A load-bearing question the TDD leaves open that must resolve before sign-off |

## 2. The tech-lead response (what the responder subagent returns to you)

The tech-lead's respond-to-review capability returns this. You parse it to score
the round. (Its authoritative definition lives in the tech-lead skill at
`~/.claude/skills/tech-lead/references/capability-respond-to-review.md`; it is
restated here so this side of the loop is self-contained.)

```json
{
  "round": <int, matches the review round being answered>,
  "tdd_path": "<path to the TDD the responder edited, if any edits were made>",
  "responses": [
    {
      "ref": "<the architect finding id being answered, e.g. '1.3'>",
      "disposition": "accept" | "challenge" | "clarify",
      "note": "<accept: what was changed in the TDD and where; challenge: the evidence-based reason the finding doesn't hold; clarify: the precise question being asked>",
      "tdd_edit": "<for accept: a short description or diff of the TDD edit applied, else null>"
    }
  ]
}
```

### How you score each disposition next round

- **`accept`** → open the TDD at the cited section and verify the edit actually
  closes the finding. Closes → `resolved`. Doesn't → `not_resolved` (re-list).
- **`challenge`** → weigh the evidence honestly against the PRD/codebase. Try to
  falsify your own finding with their evidence. If it holds → `conceded`. If your
  finding still stands → keep it; if this is the 2nd consecutive round both sides
  hold → `deadlocked` (surface to user).
- **`clarify`** → answer it if the PRD/codebase/extra docs contain the answer,
  and tell the responder. If it's genuinely a product/business decision you
  can't resolve from the inputs → surface to the user (see `review-loop.md`).

## Parsing & robustness

- Parse the responder's reply as JSON. If it isn't valid JSON, re-spawn the
  responder **once** with: "Your previous response wasn't valid JSON. Re-emit it
  as raw JSON matching the contract." If it fails again, surface to the user
  (trigger #4 in `review-loop.md`).
- Validate that every architect finding from this round has a matching `ref` in
  `responses`. A finding the responder ignored is treated as `not_resolved`.
