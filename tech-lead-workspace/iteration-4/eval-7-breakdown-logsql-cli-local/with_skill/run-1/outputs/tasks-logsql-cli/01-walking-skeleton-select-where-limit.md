### Walking skeleton: streaming SELECT/WHERE/LIMIT with CSV output

**One-liner:** Ship a minimal end-to-end `logsql` binary that streams JSON-lines through scan → filter → project → limit and writes CSV to STDOUT, proving the design's cold-start latency target on real input.

**Composes:**
- A single statically-linked Go binary invokable as `logsql [-f FILE] 'SELECT ... WHERE ... LIMIT N'` and from STDIN when no file is given.
- Streaming operator pipeline (`Next() (Row, error)`-style) wired for scan, filter, project, and limit — all O(1) memory — so the rest of the executor work plugs into a proven seam.
- Argument handling that rejects "both `-f` and STDIN provided", reads from STDIN otherwise, and emits CSV with a header row by default.
- The exit-code convention from §4.5 wired end-to-end: `0` rows emitted, `1` zero rows matched, `2` parse error, `3` runtime error. This is a new convention for the team and must be obvious from the very first release.
- A cold-start benchmark in CI on a representative small JSON-lines fixture, asserting first-row-to-STDOUT < 100ms for `SELECT * WHERE ... LIMIT 10`.
- A throwaway parser sufficient only for the subset this task exercises (single-table SELECT cols, single-predicate WHERE on equality/comparison, LIMIT) — to be replaced wholesale by the parser task; this exists only to make the skeleton walk.

**TDD sections addressed:** §3 High-level approach, §4.1 Command surface, §4.3 Executor (streaming half), §4.4 IO, §4.5 Exit codes, §6 NFR cold-start latency.

**Depends on:** none

**Acceptance criteria:**
- Binary is statically linked (no runtime install needed on a stock laptop) and runs on macOS and Linux.
- `printf '{"a":1}\n{"a":2}\n' | logsql 'SELECT a WHERE a > 1 LIMIT 10'` prints CSV with a header row and exits `0`.
- A query that matches zero rows exits `1` (and still emits the header row by default).
- A query the throwaway parser cannot handle exits `2` with `logsql: parse: ...` on STDERR; runtime errors exit `3` with `logsql: runtime: ...`.
- Passing both `-f` and piped STDIN exits `2` with a clear "choose one input source" message.
- CI benchmark fails the build if first-row-to-STDOUT exceeds 100ms on the fixture for a `SELECT * WHERE ... LIMIT 10` query.
- Streaming operators (scan, filter, project, limit) hold O(1) memory — verified by running against a fixture larger than the default 512 MB budget without breaching it.
