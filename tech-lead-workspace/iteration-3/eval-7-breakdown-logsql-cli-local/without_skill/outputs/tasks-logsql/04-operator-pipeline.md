# 04 — Operator pipeline framework

## Outcome

The `Operator` abstraction (`Next() (Row, error)` interface) and a planner that
walks a parsed AST and builds an operator pipeline: scan → filter → project →
aggregate → sort → limit.

## Why

TDD §4.3: the whole executor is defined as a pipeline of operators with a
uniform `Next()` interface. Defining the contract before implementing
individual operators lets tasks 05/06/07 proceed in parallel.

## Acceptance criteria

- `Operator` interface defined with `Next() (Row, error)` and a `Schema()`
  accessor; sentinel `io.EOF` (or equivalent) signals end of stream.
- Planner takes an AST (from task 02) and a scan operator (from task 03) and
  returns a fully-wired pipeline root, or a planning error (e.g. unknown
  column referenced) with the column name and AST location.
- Pipeline construction enforces the operator order documented in TDD §3
  (scan → filter → project → aggregate → sort → limit); illegal AST shapes
  fail at plan time, not run time.
- A stub `printPipeline` helper emits the operator tree (used by task 10's
  `--explain`).
- Unit tests cover: simple `SELECT * LIMIT 5`, `SELECT ... WHERE ...`,
  `SELECT ... GROUP BY ...`, `SELECT ... ORDER BY ... LIMIT ...`, and 3
  plan-error cases (unknown column, aggregate over non-aggregate column
  without GROUP BY, ORDER BY column not in SELECT).

## Out of scope

- Actual filter/project/aggregate/sort/limit implementations (tasks 05–07).
- CSV/JSON output (task 08).

## Dependencies

02, 03.
