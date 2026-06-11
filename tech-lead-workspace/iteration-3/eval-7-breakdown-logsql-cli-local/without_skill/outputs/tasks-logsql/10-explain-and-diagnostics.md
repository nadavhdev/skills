# 10 — `--explain`, panic handler, and diagnostics polish

## Outcome

The `--explain` flag that prints the parsed AST and operator pipeline without
running the query, plus a panic handler that prints a stable issue-report
block on unexpected crashes.

## Why

TDD §9: both are explicit diagnostic requirements for this CLI workload (the
"observability" section, reframed for a local CLI).

## Acceptance criteria

- `--explain`:
  - Prints the parsed AST in a human-readable form.
  - Prints the operator pipeline tree (uses `printPipeline` from task 04).
  - Does **not** read input or execute; exit code 0 if the query parses and
    plans; exit code 2 if it doesn't.
- Panic handler installed at `main` boundary:
  - On panic, prints a fixed-format block to STDERR with:
    `logsql version`, full command-line args (with secrets-free env), top
    stack frame, and a one-line "Please report at <URL>" pointer.
  - Exit code 3.
- Parse error messages (already produced by task 02) are surfaced via the
  CLI error formatter from task 09; integration test verifies the column
  pointer makes it to STDERR.
- Snapshot tests for `--explain` output on 3 representative queries.

## Out of scope

- Crash report uploading / telemetry — not in scope (TDD §6 security says we
  only do what the user can do locally).

## Dependencies

02, 04, 09.
