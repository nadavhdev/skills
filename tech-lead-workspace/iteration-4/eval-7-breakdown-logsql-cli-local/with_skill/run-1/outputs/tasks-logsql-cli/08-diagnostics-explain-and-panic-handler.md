### Implement diagnostics: STDERR format, `--explain`, and panic handler

**One-liner:** Standardize the diagnostic surface — the `logsql: <category>: <message>` STDERR convention, the `--explain` mode that prints AST + pipeline without executing, and the crash-time issue-report block — so incidents-about-the-tool are debuggable from a single STDERR transcript.

**Composes:**
- The `logsql: <category>: <message>` STDERR prefix applied uniformly across parse, runtime, and schema-warning paths produced by tasks 01–06. This is a service-wide concern: every operator's errors and every IO warning route through this single formatter, and inconsistency here defeats the whole convention.
- `--explain` per §9: prints the parsed AST and the assembled operator pipeline for the given query and exits without reading any input. Cross-cutting because it needs the AST shape from task 02 and the pipeline shape from tasks 01/03/04 — no single feature owns it.
- Panic handler per §9 that catches unexpected panics and prints a stable issue-report block (binary version, args used with secrets-safe redaction, top stack frames) to STDERR before exiting `3`. Service-wide because any operator can panic; a per-operator handler would diverge.
- Help text and error categories documented together so users (and the team running incident postmortems) know what each prefix means.

**TDD sections addressed:** §9 Exit codes & diagnostics, §4.5 Exit codes (consistent runtime-error surface).

**Depends on:** Implement the strict recursive-descent SQL parser, Implement GROUP BY and aggregate functions with memory budget enforcement, Implement ORDER BY with memory budget enforcement, Implement schema-on-read coercion, `--strict-schema`, and malformed-line handling, Add JSON-lines output format (`-o json`)

**Acceptance criteria:**
- Every STDERR line emitted by the binary in any error or warning condition matches `logsql: <category>: <message>` with categories drawn from a small documented set (`parse`, `runtime`, `schema`, `internal`).
- `logsql --explain 'SELECT level, COUNT(*) GROUP BY level ORDER BY COUNT(*) DESC LIMIT 5'` prints a human-readable AST and the pipeline operator list to STDOUT, reads no input, and exits `0`.
- `--explain` over a query that fails to parse exits `2` with the same parse error a real run would produce.
- A deliberately-triggered panic (test-only hook) produces an issue-report block on STDERR including version, sanitized args, and a stable stack frame; exit code is `3`.
- The panic handler does not leak environment variables or absolute file paths the user did not pass on the command line.
- The §9 STDERR convention is verified by a CI check that greps for any STDERR line not matching the prefix in the test suite's expected-failure cases.
