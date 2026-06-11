# logsql — task breakdown

I read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`
and broke it into 12 task files (plus a README) saved as a local directory at
`./tasks-logsql/`. Each task captures an outcome and acceptance criteria — no
implementation code — and is sized to be picked up by one engineer.

## Where the tasks live

```
tasks-logsql/
  README.md                         — order, dependency waves, open question
  01-scaffold.md                    — Go module, static binary, CI
  02-parser.md                      — recursive-descent SQL subset parser
  03-jsonl-input.md                 — JSON-lines streaming reader
  04-operator-pipeline.md           — Next() interface + planner
  05-streaming-ops.md               — filter, project, limit
  06-aggregate.md                   — GROUP BY + agg funcs + memory budget
  07-sort.md                        — ORDER BY + memory budget
  08-output-writers.md              — CSV (default) + JSON-lines
  09-cli-shell.md                   — flags, exit codes (0/1/2/3), errors
  10-explain-and-diagnostics.md     — --explain + panic handler
  11-nfr-benchmarks.md              — cold-start + throughput, CI-gated
  12-distribution.md                — Homebrew + internal repo, v0.1 → v0.2
```

## Suggested order (from `tasks-logsql/README.md`)

| Wave | Tasks                                                      |
|------|------------------------------------------------------------|
| 1    | 01                                                         |
| 2    | 02, 03, 09 (parallel)                                      |
| 3    | 04                                                         |
| 4    | 05, 06, 07, 08 (parallel)                                  |
| 5    | 10, 11                                                     |
| 6    | 12                                                         |

## One thing to flag before wave 4

Open Question 1 in the TDD — final v1 SQL subset (`LIKE`, `LOWER`, `UPPER`,
`COALESCE`, `CASE`; date functions deferred to v1.1) — must be signed off by
the platform team before task `02-parser` is closed. The parser strictness
decision means the documented subset *is* the contract, so this isn't
deferrable past parser merge.

## Things I deliberately did **not** create tasks for

Per TDD §2 (out of scope): JOINs, subqueries, window functions, persistent
indexes, remote sources, auth, daemon mode. Also intentionally out: spill-to-
disk for blocking operators (TDD §4.3 names this as explicitly out of v1),
Linux distro packages and Windows builds (no demand stated in the TDD).

## New convention to socialize

Exit code `1` for "query ran, zero rows matched" is a new convention introduced
by this TDD. Task `09-cli-shell.md` makes it explicit in acceptance criteria
that this must land in `--help` and the README, not as a quiet behavior change.
