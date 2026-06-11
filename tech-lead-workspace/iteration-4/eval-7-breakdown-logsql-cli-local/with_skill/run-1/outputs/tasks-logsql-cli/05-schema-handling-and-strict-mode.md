### Implement schema-on-read coercion, `--strict-schema`, and malformed-line handling

**One-liner:** Make the executor's handling of missing columns, type mismatches, and malformed JSON lines explicit and switchable between the lenient default and `--strict-schema`, so oncall users get a predictable contract on dirty log streams.

**Composes:**
- Default schema-on-read behavior per §4.1: when a referenced column is missing or the wrong type on a given row, coerce or treat as null and continue.
- `--strict-schema` flag per §4.1 and §4.4: on the first row where a referenced column is missing or the wrong type, exit `3` with a runtime error naming the column and the row's position in the stream.
- Malformed-JSON-line handling per §4.4: by default, skip the line with a warning to STDERR; under `--strict-schema`, the same condition is fatal with exit `3`.
- `--no-headers` per §4.1 for CSV output, so the tool composes into pipelines that don't expect a header row.
- Consistent STDERR diagnostic format aligned with the `logsql: <category>: <message>` convention adopted in the diagnostics task — schema warnings use the same prefix.

**TDD sections addressed:** §4.1 Command surface (`--strict-schema`, `--no-headers`), §4.4 IO (malformed-line handling), §7 Risks (schema drift mid-stream).

**Depends on:** Implement the strict recursive-descent SQL parser

**Acceptance criteria:**
- A JSON-lines stream with mixed schemas (some rows missing a column, some with a string where an integer is expected) runs to completion under default mode and exits `0` when rows match, `1` when none do.
- The same stream under `--strict-schema` exits `3` on the first offending row, with a STDERR message naming the column, the expected type, and the row position.
- A malformed JSON line in default mode emits a single STDERR warning and the query continues; under `--strict-schema`, the same line exits `3` immediately.
- `--no-headers` suppresses the CSV header row; without the flag, the header row is emitted (matching the task-01 default).
- The lenient default does not regress the task-01 cold-start benchmark (the per-row coercion overhead stays inside the latency budget).
- Behavior is documented in the help text and the README so oncall users know which mode they are in.
