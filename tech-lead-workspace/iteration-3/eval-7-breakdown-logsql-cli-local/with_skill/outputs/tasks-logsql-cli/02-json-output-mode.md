### JSON-lines output mode

**One-liner:** Add `-o json` so query results stream out as one JSON object per line, preserving the streaming flush behavior already established by the CSV writer.

**Composes:**
- `-o csv|json` flag accepted on the command line; `csv` remains the default; any other value is a parse error (exit 2) with a clear message.
- JSON-lines writer that emits one object per result row using the projection column names as keys, flushing per row for streaming operators so the first row appears with the same cold-start characteristics as CSV output.
- Type fidelity: numbers emit as JSON numbers, booleans as JSON booleans, nulls as JSON `null`, strings as JSON strings (no quoting ambiguity carried over from CSV).
- `--no-headers` documented as a no-op under `-o json` (JSON-lines has no header row), with no error raised so that scripts that always pass `--no-headers` keep working.

**TDD sections addressed:** §4.1 Command surface (`-o csv|json`), §4.4 IO (output writers, flush semantics).

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**Acceptance criteria:**
- `logsql -o json 'SELECT level, latency_ms FROM stdin WHERE status >= 500'` emits one JSON object per matching row on its own line, with keys `level` and `latency_ms`, and types preserved from the source rows.
- Streaming queries (no GROUP BY / ORDER BY) produce the first output row with no perceptible buffering delay relative to the CSV path.
- `-o yaml` (or any unsupported format) exits 2 with a STDERR message naming the accepted values.
- A query that matches zero rows under `-o json` emits no output to STDOUT and exits 1 (consistent with the exit-code contract).
