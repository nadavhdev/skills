# logsql — task breakdown

**Source TDD:** `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`
**Date of breakdown:** 2026-06-10
**Task count:** 9

## Tasks

1. `01-walking-skeleton-select-where-limit.md` — Streaming SELECT/WHERE/LIMIT CSV binary with cold-start benchmark. **Depends on:** none.
2. `02-recursive-descent-parser.md` — Strict recursive-descent parser for the documented SQL subset. **Depends on:** 01.
3. `03-group-by-and-aggregates.md` — GROUP BY and aggregate functions with memory-budget enforcement. **Depends on:** 02.
4. `04-order-by-with-memory-budget.md` — ORDER BY blocking operator with memory-budget enforcement. **Depends on:** 02.
5. `05-schema-handling-and-strict-mode.md` — Coerce/null default, `--strict-schema`, malformed-line handling, `--no-headers`. **Depends on:** 02.
6. `06-json-output-format.md` — `-o json` JSON-lines output with documented flush semantics. **Depends on:** 01.
7. `07-throughput-benchmark-and-fixture.md` — 1 GB synthetic fixture and CI throughput benchmark gating 1 GB / 60s. **Depends on:** 01, 05.
8. `08-diagnostics-explain-and-panic-handler.md` — STDERR prefix convention, `--explain`, panic handler with issue-report block. **Depends on:** 02, 03, 04, 05, 06.
9. `09-distribution-and-v0-1-rollout.md` — Homebrew + internal binary repo, signed releases, v0.1 platform-team rollout. **Depends on:** 01–08.

## Dependency graph

```
01 walking skeleton
 ├─→ 02 parser
 │    ├─→ 03 aggregates ──┐
 │    ├─→ 04 order by   ──┤
 │    └─→ 05 schema  ─────┤
 ├─→ 06 json output ──────┤
 └─→ (05 also gates) 07 throughput benchmark ──┐
                                                ├─→ 08 diagnostics ──→ 09 distribution
                                                │
       03, 04, 05, 06 ──────────────────────────┘
```

Healthy shape: 1 walking-skeleton foundation → 1 parser foundation → fan-out of 4 independent feature tasks (03, 04, 05, 06) plus 1 throughput-NFR task (07) → 1 cross-cutting diagnostics task (08) → 1 rollout task (09). Tasks 03/04/05/06 are independently picked up by separate engineers once 02 lands.

## Risk-front-loading

Task 01 (walking skeleton) deliberately retires the riskiest unknown first: whether the streaming-pipeline-in-Go design actually hits the §6 cold-start NFR (<100ms first row). If it doesn't, the design needs to change, and we want to know that in week 1, not week 5. If the team is pulled to a higher-priority incident after task 01, the binary still does useful work — `SELECT WHERE LIMIT` over JSON-lines is the most common oncall use case in the PRD.

## Open questions blocking tasks

The TDD has **1 open question** that affects no task in this breakdown directly but **gates the v0.2 broad publish** in task 09:

> §10.1 — Exact SQL subset: which string functions (`LOWER`, `LIKE`), date functions, and `CASE` are in v1. The TDD recommends shipping `LIKE`, `LOWER`, `UPPER`, `COALESCE`, `CASE` and deferring date functions to v1.1, but this needs platform-team sign-off.

Why this does **not** block tasks 02–06: the parser and executor cores in this breakdown cover only the locked-in subset from §2 (SELECT / WHERE / GROUP BY / ORDER BY / LIMIT + the six aggregates + comparison/arithmetic operators). The string/date/`CASE` functions would be additive work after platform-team sign-off. Task 02 explicitly defers them and notes the parser must be structured to accept them later.

If the platform team resolves §10 before task 09 ships, a 10th task ("Implement the agreed v1 function subset: `LIKE`, `LOWER`, `UPPER`, `COALESCE`, `CASE`") slots between 08 and 09. Do not write it until the sign-off lands — that decision is owed to the TDD, not to this breakdown.

## What to look at first

- **Task 01.** This is where the design either proves itself or doesn't. Worth a careful design review before the engineer starts coding, because if the streaming-Go-pipeline approach can't hit <100ms cold start, every task after it is wasted work.
- **Task 08 boundaries.** The cross-cutting diagnostics task touches every prior task's STDERR output. I considered folding the STDERR prefix into each feature task and keeping only `--explain` + panic handler as cross-cutting, but that risks inconsistency across features (which is the whole reason §9 exists). Worth a sanity check with whoever owns the eventual incident-postmortem flow.
- **Task 09 dependency length.** It depends on everything, which is correct for a v0.1 rollout, but it makes 09 the obvious thing to cut if a deadline tightens — the team can dogfood internally without Homebrew packaging if needed.
