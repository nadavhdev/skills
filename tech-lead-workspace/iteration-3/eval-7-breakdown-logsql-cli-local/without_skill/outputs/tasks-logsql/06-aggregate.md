# 06 — Aggregate operator with memory budget

## Outcome

A blocking aggregate operator that implements `GROUP BY` and the v1 aggregate
functions, with an enforced memory budget that produces a typed error when
exceeded.

## Why

TDD §4.3 and §7: aggregate is one of two blocking operators; the named,
enforced memory budget (default 512 MB, `--max-memory`) is a hard NFR. Memory
blow-up on large GROUP BY is the #1 risk in §7.

## Acceptance criteria

- Implements `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT col)`.
- Supports `GROUP BY` over one or more columns; supports aggregate-without-
  GROUP BY (single result row).
- Tracks accumulated memory (hash table + distinct sets) against the budget
  from the CLI flag (task 09).
- On budget exceeded: returns a typed error naming **the operator (`aggregate`)
  and the offending column / group key**, per TDD §7. Exit code 3.
- **No silent spill-to-disk in v1** — explicit, per TDD §4.3.
- Output schema is correctly inferred (group columns + aggregate result
  columns with stable names).
- Unit tests cover: each aggregate function, multi-column GROUP BY,
  COUNT(DISTINCT) on string and numeric columns, NULL handling per SQL
  standard, budget-exceeded path (assert error type, operator name, column
  name, exit code).

## Out of scope

- Sort (task 07); the aggregate output is unordered and ORDER BY happens
  downstream.
- Disk spill — explicitly out of v1.

## Dependencies

04.
