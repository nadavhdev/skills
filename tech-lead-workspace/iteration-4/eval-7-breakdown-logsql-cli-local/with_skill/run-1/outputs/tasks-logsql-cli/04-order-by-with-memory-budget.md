### Implement ORDER BY with memory budget enforcement

**One-liner:** Add the blocking sort operator behind a `--max-memory`-enforced boundary so `ORDER BY` works on bounded inputs and fails loudly on unbounded ones, never silently OOMing.

**Composes:**
- Sort operator plugged into the pipeline after aggregate and before `LIMIT`, implementing the same `Next()` interface as the other operators.
- Support for `ORDER BY <col> [ASC|DESC]` on one or more columns, on both grouped and ungrouped result sets.
- Same memory-budget contract as the aggregate operator: when the accumulated rows cross `--max-memory`, return a typed runtime error naming the `sort` operator and pointing at `--max-memory` / `LIMIT` as remediations.
- No silent spill-to-disk in v1 (per §4.3).
- Correct interaction with `LIMIT` so that `ORDER BY ... LIMIT N` produces the expected top-N result on a bounded input. (Whether to do this via a bounded heap or a full sort + truncate is a dev-expert call; the acceptance criteria specify the behavior, not the mechanism.)

**TDD sections addressed:** §2 Scope (ORDER BY), §4.3 Executor (blocking operator half), §6 NFR memory, §7 Risks (memory blow-up under blocking operators).

**Depends on:** Implement the strict recursive-descent SQL parser

**Acceptance criteria:**
- `SELECT msg, ts WHERE level='ERROR' ORDER BY ts DESC LIMIT 10` returns the 10 most recent matching rows in correct order and exits `0`.
- `ORDER BY` on multiple columns (`ORDER BY a ASC, b DESC`) sorts correctly with documented tie-breaking.
- An unbounded `ORDER BY` (no `LIMIT`) over a fixture larger than `--max-memory` exits `3` with a STDERR message that names the `sort` operator and the budget value; the host does not OOM.
- An `ORDER BY ... LIMIT N` query over the same fixture succeeds within the memory budget for reasonable N, demonstrating that the top-N path does not require holding the full input.
- Streaming queries that don't use `ORDER BY` retain O(1) memory; the task-01 cold-start benchmark still passes.
- The behavior composes correctly with aggregate: `SELECT level, COUNT(*) GROUP BY level ORDER BY COUNT(*) DESC LIMIT 5` produces the expected ranked result.
