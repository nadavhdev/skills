### Implement GROUP BY and aggregate functions with memory budget enforcement

**One-liner:** Add the blocking aggregate operator (`GROUP BY` plus `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT)`) and enforce the named memory budget so an over-budget aggregation fails with a typed, actionable error instead of OOMing the laptop.

**Composes:**
- Aggregate operator implementing the same `Next()` interface as the streaming operators, so it plugs into the existing pipeline behind `WHERE` and in front of `ORDER BY` / `LIMIT`.
- All six aggregates from §2: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT)`. `COUNT(DISTINCT)` honors the same memory budget as `GROUP BY` because its working set is the same shape.
- Enforcement of the `--max-memory` budget (default 512 MB per §6): when accumulated state crosses the budget, the operator returns a runtime error naming the operator and the offending column/grouping, not a generic OOM.
- No silent spill-to-disk in v1 (explicit non-goal from §4.3) — the failure is loud and steers the user to `--max-memory` or to adding a `LIMIT`, per §7.

**TDD sections addressed:** §2 Scope (aggregates), §4.3 Executor (blocking operator half), §6 NFR memory, §7 Risks (memory blow-up on large GROUP BY).

**Depends on:** Implement the strict recursive-descent SQL parser

**Acceptance criteria:**
- `SELECT level, COUNT(*) GROUP BY level` over a JSON-lines stream produces correct per-group counts and exits `0`; zero matching rows exits `1`.
- Each of `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT col)` produces correct results on a fixture with known expected outputs, including mixed-type columns under default coerce/null semantics.
- A `GROUP BY` whose accumulated state exceeds `--max-memory` exits `3` with a STDERR message that names the operator (`aggregate`) and the budget value; the message hints at `--max-memory` and `LIMIT` as remediations.
- Setting `--max-memory` to a smaller value lowers the threshold at which the typed memory error fires; setting it larger raises it — confirmed by the same fixture used in both directions.
- Streaming `SELECT WHERE LIMIT` queries (no aggregate) retain O(1) memory and the cold-start benchmark from task 01 still passes.
- The binary does not OOM the host under any of the above scenarios — every memory exhaustion path is the typed error, not a kernel kill.
