Read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`. Covers a Go single-binary CLI that streams JSON-lines through a parsed SQL-subset pipeline (scan→filter→project→aggregate→sort→limit) with an enforced memory budget, CSV/JSON output, and a new 0/1/2/3 exit-code convention. 1 open question noted (final SQL function subset — `LIKE`, `LOWER`, `CASE`, dates). Local-directory destination confirmed per your request.

## Breakdown

Wrote 9 tasks to `./tasks-logsql-cli/` plus a README index. Task files are numbered in dependency order:

1. **`01-walking-skeleton-select-where-limit.md`** — Streaming SELECT/WHERE/LIMIT binary with CSV output, exit-code convention wired, cold-start benchmark. Deliberately first to retire the riskiest unknown: whether the Go streaming pipeline hits the §6 <100ms cold-start NFR.
2. **`02-recursive-descent-parser.md`** — Strict recursive-descent parser for the locked-in SQL subset, replacing the skeleton's throwaway parser. String/date/`CASE` functions explicitly deferred until §10 is resolved.
3. **`03-group-by-and-aggregates.md`** — GROUP BY plus the six aggregates with `--max-memory` enforcement and typed memory errors (no silent spill).
4. **`04-order-by-with-memory-budget.md`** — Sort operator with the same memory-budget contract; correct interaction with LIMIT.
5. **`05-schema-handling-and-strict-mode.md`** — Default coerce/null, `--strict-schema`, malformed-line warnings, `--no-headers`.
6. **`06-json-output-format.md`** — `-o json` with documented streaming/blocking flush semantics.
7. **`07-throughput-benchmark-and-fixture.md`** — 1 GB synthetic JSON-lines fixture and CI benchmark gating the §6 1 GB / 60s throughput NFR.
8. **`08-diagnostics-explain-and-panic-handler.md`** — Cross-cutting `logsql: <category>: <message>` STDERR convention, `--explain`, and panic handler with stable issue-report block. Justified as cross-cutting because each bullet covers service-wide concerns no single feature owns.
9. **`09-distribution-and-v0-1-rollout.md`** — Homebrew formula, internal binary repo, signed static releases, §8 staged rollout to platform-team volunteers.

## Destination

`./tasks-logsql-cli/` in the working directory — 9 task markdown files plus `README.md` (10 files total, within the ≤13 cap).

## Shape

1 walking-skeleton foundation → 1 parser foundation → fan-out of 4 independent feature tasks (03, 04, 05, 06) → 1 throughput-NFR task → 1 cross-cutting diagnostics task → 1 rollout task. After task 02 lands, three engineers can pick up 03/04/05 in parallel. If the team gets pulled to an incident after task 01, the binary still does the most common oncall use case (`SELECT WHERE LIMIT` over JSON-lines).

## Open questions

**1 open question** in the TDD (§10.1 — exact SQL function subset, recommendation: `LIKE`, `LOWER`, `UPPER`, `COALESCE`, `CASE`, defer dates). It does **not** block any task in this breakdown — every task covers only the locked-in subset from §2 and §4, and task 02 is explicitly structured to accept the string functions later. It does **gate the v0.2 broad publish** in task 09. If platform-team sign-off lands before task 09 ships, slot a 10th task ("Implement the agreed v1 function subset") between 08 and 09. Do not write that task until the decision is in — that's a call the TDD owes the breakdown, not the other way around.

## What to look at first

- **Task 01** — the design either proves itself here or it doesn't. Worth a quick design review before coding starts; everything after it depends on the streaming-Go-pipeline approach actually hitting <100ms cold start.
- **Task 08 boundaries** — I kept the STDERR prefix in the cross-cutting task rather than folding it into each feature task, because inconsistency across features defeats the §9 convention. Worth a sanity check with whoever owns incident postmortems.
- **Task 09** — depends on everything; it's the obvious cut if the deadline tightens. The team can dogfood without Homebrew packaging if needed.
