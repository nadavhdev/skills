# Task breakdown: logsql CLI

**Source TDD:** `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`
**Date of breakdown:** 2026-06-10
**Task count:** 8 deliverable tasks (1 additional task blocked — see below)

## Summary

`logsql` is a Go CLI that reads JSON-lines from STDIN or a file, runs a
SQL-subset query, and writes CSV or JSON-lines to STDOUT. The breakdown is
shaped as a walking skeleton (task 1) that ships a usable
`SELECT / WHERE / LIMIT` flow end-to-end on day one, followed by a fan-out
of independent feature tasks (output mode, schema handling, GROUP BY,
ORDER BY), then cross-cutting tasks for diagnostics, performance, and
distribution.

The walking skeleton intentionally folds in the new exit-code convention
(`1` = zero rows matched) because that contract has to be set the moment
any query can run; it cannot be retrofitted as a separate task without
breaking pipelines that already rely on it.

## Tasks (in dependency order)

| # | File | One-liner | Depends on |
|---|------|-----------|------------|
| 01 | `01-walking-skeleton-select-where-limit.md` | Ship the smallest end-to-end binary that parses `SELECT ... WHERE ... LIMIT N`, streams JSON-lines from STDIN or `-f FILE`, and writes CSV to STDOUT — with the full exit-code contract wired in. | none |
| 02 | `02-json-output-mode.md` | Add `-o json` so query results stream out as one JSON object per line. | 01 |
| 03 | `03-schema-on-read-and-strict-mode.md` | Define lenient coercion / null behavior, malformed-line warnings, and the `--strict-schema` opt-in. | 01 |
| 04 | `04-group-by-and-aggregates.md` | Add the blocking aggregate operator, the six built-in aggregate functions, and the `--max-memory` budget. | 01 |
| 05 | `05-order-by-under-memory-budget.md` | Add the blocking sort operator with multi-column ASC/DESC support under the same `--max-memory` budget. | 01, 04 |
| 06 | `06-explain-and-panic-handler.md` | `--explain` query-plan dump plus a stable panic-handler issue-report block. | 01 |
| 07 | `07-performance-benchmarks-in-ci.md` | Wire cold-start (<100 ms) and throughput (1 GB / 60 s) NFR targets into CI as hard-fail benchmarks. | 01, 02, 04, 05 |
| 08 | `08-distribution-homebrew-and-binary.md` | Homebrew formula and internal binary download, plus the staged v0.1 → v0.2 rollout. | 01 |

## Dependency graph

```
                       01 (walking skeleton)
                        |
        +-------+-------+-------+-------+-------+
        |       |       |       |       |       |
       02      03      04      06      08      (07 below)
                       |
                       05

  07 (benchmarks) ← 01, 02, 04, 05
```

Healthy fan-out shape: task 1 is the only true bottleneck. Tasks 2, 3, 4,
6, and 8 can be picked up in parallel by separate engineers once 1 lands;
task 5 follows task 4; task 7 lands last because it benchmarks the full
query surface.

## TDD sections that did NOT become tasks (and why)

- **§1 Problem & context** — context only.
- **§2 Scope** — out-of-scope items (JOINs, subqueries, window functions,
  persistent indexes, remote sources, auth, daemon mode) are explicitly
  not work for this breakdown.
- **§5 Alternatives considered** — context only.
- **§6 NFRs** — memory and cold-start / throughput NFRs ride as
  acceptance criteria on the feature tasks they constrain (memory budget
  on tasks 4 and 5; cold-start and throughput on task 7); they are not
  their own tasks per the breakdown rules.
- **§7 Risks & failure modes** — informs acceptance criteria on tasks 3
  (schema drift) and 4 (memory blow-up on GROUP BY); not its own task.

## Blocked: string functions + `CASE`

**Blocked by open question in TDD §10:**

> Exact SQL subset. The parser strictness decision means the documented
> subset is the contract. We need a final review of which functions /
> operators are in: e.g. string functions (`LOWER`, `LIKE`), date
> functions, `CASE`. Recommend: ship v1 with `LIKE`, `LOWER`, `UPPER`,
> `COALESCE`, and `CASE`; defer date functions to v1.1. Needs
> platform-team sign-off.

A 9th task — "String functions (`LIKE`, `LOWER`, `UPPER`, `COALESCE`) and
`CASE` expressions" — is **not written** in this breakdown because the
TDD's parser-strictness decision means the documented subset is the
contract: shipping any of these functions locks the team to supporting
them at the strict-parser level. The TDD names a recommendation but
explicitly flags that it needs platform-team sign-off.

**Action:** get the open question resolved (sign-off on the recommended
list, or an adjusted list), then re-run the breakdown to write the
string-function task with the final scope. The v0.2 broad-publish in
task 8 is gated on this same resolution.

## What to look at first

- **Task 1 (walking skeleton)** carries the most design weight in this
  breakdown — it bakes in the new exit-code convention, the streaming
  pipeline interface, and the STDERR error format that every later task
  reuses. Get it right and the rest are mechanical.
- **Task 4 (GROUP BY + memory budget)** is where the design's "no silent
  spill-to-disk" stance gets tested for real. If the team disagrees with
  that stance in practice, the TDD needs to change before this task
  ships, not after.
- **Task 7 (benchmarks)** depends on tasks 1, 2, 4, 5 and so will land
  late. If the team wants earlier signal that the cold-start NFR is on
  track, consider splitting an "ad-hoc cold-start benchmark on task 1"
  off as a sub-acceptance-criterion of task 1 rather than waiting for
  task 7.
