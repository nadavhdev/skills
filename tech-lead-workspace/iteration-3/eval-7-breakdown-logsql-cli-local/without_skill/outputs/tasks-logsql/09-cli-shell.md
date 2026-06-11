# 09 — CLI surface, flags, and exit codes

## Outcome

The `logsql` CLI entrypoint with the full v1 flag set, exit code conventions,
and error formatting. This is what wires the parser, scan, planner, operators,
and output writers together end-to-end.

## Why

TDD §4.1, §4.5, §9: the command surface, exit codes (0/1/2/3), and the
`logsql: <category>: <message>` STDERR prefix are part of the user contract.
Exit code 1 for "zero rows matched" is a **new convention** introduced by this
TDD and must be implemented and documented deliberately.

## Acceptance criteria

- Flags implemented per TDD §4.1:
  - `-f FILE` (mutually exclusive with STDIN; error if both)
  - `-o csv|json` (default csv)
  - `--max-memory SIZE` (default 512 MB; parses `512MB`, `1GB`, etc.)
  - `--no-headers`
  - `--strict-schema`
  - Positional: the SQL query string (required)
- Exit codes per TDD §4.5:
  - `0` — query ran, output produced (>=1 row written).
  - `1` — query ran, **zero rows matched**.
  - `2` — parse error (any error from task 02).
  - `3` — runtime error (memory budget, IO, type mismatch under
    `--strict-schema`).
- The zero-rows exit-1 convention is documented in `--help` and the user-
  facing README, and called out as new behavior (not a bug if a user is
  surprised).
- All errors go to STDERR with the `logsql: <category>: <message>` prefix.
  Categories: `parse`, `plan`, `runtime`, `io`, `warn`.
- A row counter wraps the output writer to decide between exit 0 and exit 1.
- End-to-end tests for each exit code path.

## Out of scope

- `--explain` and panic handler (task 10).

## Dependencies

01. Can be developed in parallel with 02/03 using stubs; final wire-up
happens after 04 and 08.
