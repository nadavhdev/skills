# 07 — Sort operator with memory budget

## Outcome

A blocking sort operator implementing `ORDER BY col1 [ASC|DESC], col2 ...` with
the same enforced memory budget contract as aggregate.

## Why

TDD §4.3: the second of the two blocking operators. Shares the budget contract
and "no silent spill" rule with aggregate (§7).

## Acceptance criteria

- Supports multi-column `ORDER BY` with ASC/DESC per column.
- NULL ordering matches a documented convention (recommend NULLS LAST for ASC,
  NULLS FIRST for DESC — document the choice in the user docs).
- Tracks accumulated memory against the budget from task 09.
- On budget exceeded: typed error naming the operator (`sort`) and the column
  list being sorted. Exit code 3.
- Stable sort across multi-column keys.
- Begins emitting rows only after upstream EOF (blocking behavior is explicit
  in tests).
- Unit tests cover: single-column ASC and DESC, multi-column mixed direction,
  NULL ordering, type-mismatched values in the sort column under default and
  strict modes, budget-exceeded path.

## Out of scope

- Spill-to-disk — explicitly out of v1.
- Top-K optimization for `ORDER BY ... LIMIT N` — fine to defer to v1.1
  unless NFR benchmarks (task 11) say otherwise.

## Dependencies

04.
