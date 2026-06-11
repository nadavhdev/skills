Read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`. Covers a Go CLI for SQL-subset queries over JSON-lines logs: hand-written recursive-descent parser, streaming pipeline (scan → filter → project → aggregate → sort → limit), memory-bounded blocking operators, a new exit-code convention, and Homebrew distribution. 1 open question noted (the final SQL-subset for v1 — string functions and `CASE`). Wrote tasks to the local directory.

## Destination

Local directory: `./tasks-logsql-cli/` in the working dir, one markdown file per task plus a `README.md` index. 8 deliverable task files + 1 README + 1 blocked task documented in the README — 9 files total, well inside the 13-file cap.

## Shape

Walking skeleton in task 1 (ships `SELECT / WHERE / LIMIT` end-to-end with the full exit-code contract wired in), then a fan-out of 5 independent feature tasks (output mode, schema handling, GROUP BY, ORDER BY, diagnostics), then two cross-cutting tasks (benchmarks, distribution) that land last:

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

Three engineers can pick up tasks 2/3/4 in parallel as soon as 1 lands. Task 1 is the only bottleneck.

## Blocked task (not written)

A 9th task — **String functions (`LIKE`, `LOWER`, `UPPER`, `COALESCE`) and `CASE`** — is **blocked** by TDD §10. The TDD's parser-strictness decision means the documented subset is the contract, and the TDD explicitly says the final subset needs platform-team sign-off. There's a recommendation in the TDD, but until it's signed off, writing the task would lock the team to a list the design hasn't committed to. Surfaced in the README with the open question quoted; needs the platform team to resolve before this work can start. The v0.2 broad-publish in task 8 is gated on the same resolution.

## What to look at first

- **Task 1** is the heaviest task in this breakdown — it sets the exit-code convention (including the new `exit 1` = zero rows matched), the streaming operator interface, and the STDERR error format that every later task reuses. Worth a careful first review.
- **Task 4 (GROUP BY)** is where the TDD's "no silent spill-to-disk in v1" stance gets tested in practice. If the team disagrees with that stance, the TDD should change before this task ships, not after.
- **Task 7 (benchmarks)** lands late because it needs the full query surface. If you want earlier signal that the cold-start NFR is on track, consider folding an ad-hoc cold-start check into task 1 as an extra acceptance criterion rather than waiting for task 7.

## Files

- `./tasks-logsql-cli/README.md` — index, dependency graph, blocked task, TDD-section coverage
- `./tasks-logsql-cli/01-walking-skeleton-select-where-limit.md`
- `./tasks-logsql-cli/02-json-output-mode.md`
- `./tasks-logsql-cli/03-schema-on-read-and-strict-mode.md`
- `./tasks-logsql-cli/04-group-by-and-aggregates.md`
- `./tasks-logsql-cli/05-order-by-under-memory-budget.md`
- `./tasks-logsql-cli/06-explain-and-panic-handler.md`
- `./tasks-logsql-cli/07-performance-benchmarks-in-ci.md`
- `./tasks-logsql-cli/08-distribution-homebrew-and-binary.md`
