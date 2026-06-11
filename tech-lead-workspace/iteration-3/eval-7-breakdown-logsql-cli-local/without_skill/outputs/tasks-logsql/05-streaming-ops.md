# 05 — Streaming operators: filter, project, limit

## Outcome

Implementations of the three streaming operators that hold O(1) memory: filter
(WHERE), project (SELECT column list and scalar expressions), and limit.

## Why

TDD §4.3: streaming operators pass rows through one at a time. These three are
on the hot path for the dominant use case (`SELECT * WHERE ... LIMIT 10`
during an incident — see NFR cold-start in §6).

## Acceptance criteria

- **Filter:** evaluates the WHERE expression for each row; passes rows where
  the result is `TRUE`; drops `FALSE` and `NULL` (SQL-standard three-valued
  logic).
- **Project:** evaluates each SELECT-list expression per row; supports `*`
  expansion against the current schema; supports the scalar functions decided
  in Open Question 1.
- **Limit:** stops the upstream after N rows by returning EOF.
- Each operator holds O(1) state — verified by a streaming test that runs 1M
  synthetic rows through each operator with bounded heap growth.
- Type errors during expression evaluation:
  - Default: coerce or null per TDD §4.4.
  - With `--strict-schema`: typed runtime error → exit code 3.
- Unit tests for each operator: happy path, NULL handling, type-mismatch
  default + strict, empty upstream.

## Out of scope

- Aggregate (task 06), sort (task 07).
- Output writers (task 08).

## Dependencies

04.
