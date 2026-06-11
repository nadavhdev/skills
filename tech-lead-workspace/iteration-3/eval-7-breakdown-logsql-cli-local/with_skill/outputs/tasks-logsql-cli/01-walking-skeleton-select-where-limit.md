### Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**One-liner:** Ship the smallest end-to-end `logsql` binary that parses `SELECT ... WHERE ... LIMIT N`, streams JSON-lines from STDIN or `-f FILE`, and writes CSV to STDOUT.

**Composes:**
- Single statically-linked Go binary invokable as `logsql [-f FILE] 'SELECT ... WHERE ... LIMIT N'`, with STDIN as the default input and `-f FILE` as the alternative; supplying both is rejected with a usage error.
- Hand-written recursive-descent parser bootstrap covering identifiers, literals, comparison and boolean operators, `SELECT` projection list (including `*`), `WHERE` predicate, and `LIMIT`. AST output; no external SQL parser.
- Streaming executor pipeline: scan (JSON-lines decoder) → filter (WHERE) → project (SELECT) → limit (LIMIT). All four operators implement the `Next() (Row, error)` interface and hold O(1) memory.
- CSV writer (default output) with a header row derived from the projection, flushing per row as the streaming pipeline produces them.
- Exit-code contract wired in: `0` on successful execution with at least one row emitted, `1` on successful execution with zero rows matched (new convention — documented in `--help` and README), `2` on parse error with a column-pointing message, `3` on runtime error.
- STDERR error format `logsql: <category>: <message>` established for parse and runtime errors so downstream tasks reuse it.

**TDD sections addressed:** §3 High-level approach, §4.1 Command surface, §4.2 Parser, §4.3 Executor (streaming operators), §4.4 IO (JSON-lines input, CSV output), §4.5 Exit codes, §9 Exit codes & diagnostics (error prefix).

**Depends on:** none

**Acceptance criteria:**
- `logsql 'SELECT level, msg FROM stdin WHERE level = "error" LIMIT 5' < access.jsonl` prints up to 5 matching rows as CSV with a header row, then exits 0.
- The same invocation against an input where no rows match prints only the header row and exits 1; the exit code distinguishes "no match" from "parse/runtime error" in shell pipelines.
- A query with an unknown construct (e.g. a JOIN clause) exits 2 with a STDERR message of the form `logsql: parse: <message>` that points at the column in the query string where parsing failed.
- A query referencing the input via both `-f FILE` and piped STDIN exits 2 with a usage error; neither input is read.
- Memory usage stays within a small constant bound regardless of input size for streaming-only queries (verified by running the binary against a multi-GB synthetic file with `LIMIT 10` and observing RSS does not grow with input size).
- `--help` documents the four exit codes including the new `1` = zero rows convention.
