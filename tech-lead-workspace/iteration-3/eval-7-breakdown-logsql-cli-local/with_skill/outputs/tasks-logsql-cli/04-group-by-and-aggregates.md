### GROUP BY with `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT)` under a memory budget

**One-liner:** Add the blocking aggregate operator, the six built-in aggregate functions, and the named `--max-memory` budget that protects the laptop from a runaway GROUP BY during an incident.

**Composes:**
- Parser extended to accept `GROUP BY <expr-list>`, aggregate function calls in the projection, and `COUNT(DISTINCT <expr>)`. Aggregates referenced outside an aggregate context (e.g. a non-grouped column in the projection) are a parse error.
- Six aggregates implemented: `COUNT(*)`, `COUNT(expr)` (counts non-null), `SUM`, `AVG`, `MIN`, `MAX`, and `COUNT(DISTINCT expr)`. Semantics match standard SQL on the documented value-type set.
- Aggregate operator implementing `Next()` as a blocking operator: consumes the full upstream, builds groups in memory, then yields one row per group.
- `--max-memory SIZE` flag (default 512 MB) parsed at startup; size accepts a number with optional `K`/`M`/`G` suffix. The aggregate operator tracks its own group-table size and, when the budget is exceeded, returns a typed runtime error naming `aggregate` as the operator that blew the budget and the approximate group count at the time of failure.
- No silent spill-to-disk in v1 — explicitly per TDD §4.3; the error message tells the user to re-run with `--max-memory` raised or to add a tighter `WHERE` / `LIMIT`.
- Cooperation with `LIMIT`: `LIMIT` applies after aggregation (per SQL semantics), so a `LIMIT 10` does not bound aggregate memory.

**TDD sections addressed:** §3 High-level approach (blocking operators), §4.1 Command surface (`--max-memory`), §4.2 Parser (GROUP BY, aggregates), §4.3 Executor (aggregate operator, memory budget), §6 NFR: memory (512 MB default), §7 Risks (memory blow-up on large GROUP BY).

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**Acceptance criteria:**
- `logsql 'SELECT level, COUNT(*) FROM stdin GROUP BY level'` against a multi-level log stream produces one row per distinct `level` with the correct count, exits 0.
- `SELECT COUNT(DISTINCT user_id) FROM stdin WHERE status = 500` returns a single row with the distinct count.
- A `GROUP BY` query that would build a group table larger than the configured `--max-memory` exits 3 with a STDERR message naming `aggregate` as the offending operator and the configured budget; no partial output appears on STDOUT.
- The same query succeeds when re-run with `--max-memory 2G` (assuming the laptop has the headroom).
- `SELECT COUNT(*) FROM stdin` against a 10M-row stream returns a single correct count using memory bounded by group cardinality (one group), not by row count.
- Referencing a non-grouped, non-aggregated column in the projection (`SELECT user_id, COUNT(*) FROM stdin GROUP BY level`) exits 2 with a parse error.
