# logsql — Task breakdown

Source TDD: `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-logsql-cli.md`

These are the deliverable tasks for building `logsql` v0.1 → v0.2. Each task
file describes an outcome and acceptance criteria, not implementation code.

## Suggested order

The numbering encodes a dependency-respecting order. Items in the same row can
be picked up in parallel by different engineers; items in a later row depend on
earlier rows.

| Wave | Tasks                                  | Notes                              |
|------|----------------------------------------|------------------------------------|
| 1    | 01-scaffold                            | unblocks everything                |
| 2    | 02-parser, 03-jsonl-input, 09-cli-shell | independent foundations           |
| 3    | 04-operator-pipeline                   | depends on 02 (AST) + 03 (rows)    |
| 4    | 05-streaming-ops, 06-aggregate, 07-sort, 08-output-writers | depend on 04        |
| 5    | 10-explain-and-diagnostics, 11-nfr-benchmarks | depend on a working end-to-end |
| 6    | 12-distribution                        | last; ships v0.1 to volunteers     |

## Open question to resolve before wave 4

Open question 1 in the TDD (final v1 SQL subset — `LIKE`, `LOWER`, `UPPER`,
`COALESCE`, `CASE`; date functions deferred). This must be resolved with the
platform team before task `02-parser` is closed, because the parser strictness
decision means the documented subset *is* the contract.

## Out of scope (do not file tasks for these)

JOINs, subqueries, window functions, persistent indexes, remote sources, auth,
daemon mode. Per TDD section 2.
