### Implement the strict recursive-descent SQL parser

**One-liner:** Replace the walking skeleton's throwaway parser with the full recursive-descent parser for the documented SQL subset, producing an AST and column-pointing errors with no external SQL parser dependency.

**Composes:**
- Hand-written recursive-descent parser covering the locked-in grammar: `SELECT` projection lists (columns, aliases, aggregate calls), `WHERE` with boolean/comparison/arithmetic expressions, `GROUP BY`, `ORDER BY` (ASC/DESC), `LIMIT`, and the aggregate functions named in §2 (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT ...)`).
- Strict-mode behavior per §3: any construct outside the documented subset is a parse error, never a permissive best-effort. The error message points to the column in the query string.
- A well-defined AST consumed by the executor — stable enough that aggregate, sort, and diagnostics tasks can build on it independently.
- Zero external SQL parser dependencies (per §5 alternatives considered — this is a hard design decision, not preference).
- The string / date / `CASE` function surface from §10 is **deliberately deferred** — the parser must be structured so adding `LIKE` / `LOWER` / `UPPER` / `COALESCE` / `CASE` later is local, but they are not implemented in this task.

**TDD sections addressed:** §3 High-level approach (parser strictness), §4.2 Parser, §5 Alternatives considered.

**Depends on:** Walking skeleton: streaming SELECT/WHERE/LIMIT with CSV output

**Acceptance criteria:**
- Every in-scope construct from §2 parses to an AST without falling back to the throwaway parser.
- Any construct outside the documented subset (e.g. `JOIN`, subquery, window function, undocumented function name) produces exit code `2` with a parse error message that names the column position in the query string.
- A malformed query like `SELECT a, FROM ...` reports the offending column position, not just "syntax error".
- No third-party SQL parser is in the dependency graph (verified by inspecting the module manifest).
- The walking skeleton's CI benchmark still passes (cold-start < 100ms) after the parser swap — the new parser does not regress first-row latency.
- All in-scope queries an oncall engineer might run round-trip through parse → AST → execute, including queries that combine WHERE, GROUP BY, ORDER BY, and LIMIT.
