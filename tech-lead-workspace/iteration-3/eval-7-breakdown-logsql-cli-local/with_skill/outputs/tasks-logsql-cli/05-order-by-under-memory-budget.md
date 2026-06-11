### ORDER BY operator under the memory budget

**One-liner:** Add the blocking sort operator with multi-column ASC/DESC support, sharing the `--max-memory` budget machinery from the aggregate task.

**Composes:**
- Parser extended to accept `ORDER BY <expr> [ASC|DESC] [, <expr> [ASC|DESC]]*`, including ordering by aggregate outputs and projected aliases.
- Sort operator implementing `Next()` as a blocking operator: consumes the full upstream, sorts in memory, then yields rows in order.
- Memory accounting integrates with the same `--max-memory` budget defined for aggregates; when the in-memory buffer exceeds the budget, the operator returns a typed runtime error naming `sort` as the offending operator and the approximate row count at the time of failure.
- Null ordering: nulls sort last under ASC, first under DESC (documented in `--help`).
- Stable sort: rows with equal sort keys preserve input order, so a `ORDER BY level, timestamp` query is deterministic when timestamps tie.
- Composition with `LIMIT`: `ORDER BY ... LIMIT N` is the common "top-N" shape — the sort still materializes all upstream rows in v1 (no top-N optimization); this is named in the task so a future task can revisit if benchmarks warrant.

**TDD sections addressed:** §3 High-level approach (blocking operators), §4.2 Parser (ORDER BY), §4.3 Executor (sort operator, memory budget), §6 NFR: memory.

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV, GROUP BY with `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT)` under a memory budget

**Acceptance criteria:**
- `logsql 'SELECT timestamp, latency_ms FROM stdin ORDER BY latency_ms DESC LIMIT 10'` returns the 10 slowest rows in descending order; ties broken by input order.
- `ORDER BY level, timestamp` produces output sorted primarily by `level` ascending, then by `timestamp` ascending.
- `ORDER BY` combined with `GROUP BY` (`SELECT level, COUNT(*) FROM stdin GROUP BY level ORDER BY 2 DESC`) returns groups in descending count order.
- A sort whose materialized buffer exceeds `--max-memory` exits 3 with a STDERR message naming `sort` as the offending operator and the configured budget; no partial sorted output appears on STDOUT.
- Null values appear at the end under ASC and at the start under DESC; this is documented in `--help`.
