### `--explain` and panic-handler diagnostics

**One-liner:** Replace the "observability" surface of a server workload with the two things a local CLI actually needs during an incident: a query-plan dump and a stable crash report.

**Composes:**
- `--explain` flag that parses the query, prints the AST and the resolved operator pipeline (scan → filter → project → aggregate → sort → limit, with only the operators actually instantiated for the query), and exits 0 without running the query.
- The `--explain` output is human-readable and stable enough to paste into an incident channel — same query produces the same output across runs of the same binary version.
- Panic handler installed in `main` that catches any unrecovered runtime panic and prints a stable issue-report block to STDERR with: `logsql` version, the exact argv received, and a stack frame summary. The process then exits with code 3.
- The issue-report block has a clearly delimited start and end (e.g. `---logsql-issue-report---`) so a user can copy-paste it without grabbing surrounding noise.
- The panic handler is mute under normal exits — it only fires on a genuine panic. Expected runtime errors (memory budget, parse error, IO error) flow through the normal error path established in the walking skeleton.

**TDD sections addressed:** §9 Exit codes & diagnostics (`--explain`, crash reports).

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**Acceptance criteria:**
- `logsql --explain 'SELECT level, COUNT(*) FROM stdin GROUP BY level ORDER BY 2 DESC LIMIT 5'` prints the parsed AST and the pipeline `scan → filter? → project → aggregate → sort → limit` (with the filter elided since there is no `WHERE`), reads no input from STDIN, and exits 0.
- `--explain` against an invalid query exits 2 with a parse error, same as without `--explain`.
- A deliberately-triggered panic (e.g. via a test build that panics in the executor) produces a STDERR block delimited by `---logsql-issue-report---` lines containing `version=`, `argv=`, and a stack frame summary, then exits 3.
- Normal runtime errors (memory budget exceeded, malformed line under `--strict-schema`) do not produce the issue-report block — they continue to print the `logsql: runtime: <category>: <detail>` format.
- The `--explain` output for the same query and binary version is byte-identical across runs (no timestamps, no random IDs).
