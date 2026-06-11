### Schema-on-read coercion, malformed lines, and `--strict-schema`

**One-liner:** Define how the executor handles missing columns, type mismatches, and malformed JSON-lines under the default lenient mode and under `--strict-schema`.

**Composes:**
- Default lenient behavior: a reference to a column not present in a given row produces a null in that row's evaluation; a referenced value of the wrong type is coerced where unambiguous (e.g. numeric string ↔ number for comparisons) and produces a null otherwise.
- Malformed JSON-lines: a line that fails to parse as JSON is skipped under the default mode, with a single STDERR warning of the form `logsql: warn: skipped malformed line N` (line numbers are 1-based, counted as read).
- `--strict-schema` flag that upgrades both failures to fatal: the first missing-column reference, the first type-coercion failure, or the first malformed line ends the query with exit code 3 and a STDERR message of the form `logsql: runtime: <category>: <detail>` naming the line number and the offending column.
- Decision is per-evaluation, not global: a row that does not reference the missing column continues to flow through the pipeline normally even under `--strict-schema`.
- Behavior is identical under CSV and JSON output modes — output mode does not affect schema handling.

**TDD sections addressed:** §4.1 Command surface (`--strict-schema`), §4.4 IO (malformed line handling), §7 Risks & failure modes (schema drift mid-stream).

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**Acceptance criteria:**
- Querying `SELECT user_id FROM stdin WHERE user_id IS NOT NULL` against an input where some rows lack `user_id` returns only rows where the column is present and non-null, with no STDERR warnings in the default mode.
- A query that references a numeric column against rows where the value is a numeric string returns the same results as if the values were numbers (e.g. `WHERE latency_ms > 100` matches `"latency_ms": "250"`).
- A malformed JSON line in the middle of an otherwise valid stream produces a single STDERR warning naming the line number, and the query completes successfully against the remaining lines.
- Under `--strict-schema`, the same malformed input exits 3 at the malformed line with a STDERR message; rows after the failure are not processed and not emitted.
- Under `--strict-schema`, a query referencing a column not present in some rows exits 3 on the first such row, naming the column and the line number.
