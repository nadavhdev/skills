### Add JSON-lines output format (`-o json`)

**One-liner:** Add `-o json` so query results can flow into downstream JSON-consuming tools without an intermediate CSV-to-JSON hop, with the same streaming flush semantics as CSV output.

**Composes:**
- A JSON-lines writer behind the same output interface as the CSV writer, selected by `-o json` per §4.1; default remains CSV.
- Streaming-flush semantics per §4.4: rows from streaming operators (scan/filter/project/limit) flush as they arrive; rows from blocking operators (aggregate/sort) flush once the operator completes.
- Consistent encoding for nulls, numbers, and strings between CSV and JSON output for the same query and input — same query, different `-o`, same logical rows.
- `--no-headers` is a no-op for JSON output (header concept doesn't apply) and the help text says so explicitly to avoid confusion.

**TDD sections addressed:** §2 Scope (CSV and JSON output, piping to other tools), §4.1 Command surface (`-o csv|json`), §4.4 IO (output).

**Depends on:** Walking skeleton: streaming SELECT/WHERE/LIMIT with CSV output

**Acceptance criteria:**
- `logsql -o json 'SELECT a, b WHERE a > 1 LIMIT 5'` emits one valid JSON object per line, parseable by `jq -c .` without errors.
- A streaming query writes the first row to STDOUT within the same cold-start budget as the CSV path (task-01 benchmark passes under `-o json` too).
- For a query like `SELECT level, COUNT(*) GROUP BY level`, JSON output appears only after the aggregate completes — matching the documented blocking-flush semantics.
- Null values, integers, floats, and strings round-trip without lossy stringification (a numeric `1` in the input stays a JSON number, not a string).
- Exit-code behavior (`0` / `1` / `2` / `3`) is identical between `-o csv` and `-o json` for the same query and input.
- Passing an unknown value to `-o` (e.g. `-o yaml`) exits `2` with a parse-time error message that lists the supported formats.
