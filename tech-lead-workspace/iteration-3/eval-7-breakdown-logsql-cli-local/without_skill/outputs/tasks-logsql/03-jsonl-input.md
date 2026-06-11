# 03 — JSON-lines input reader

## Outcome

A streaming JSON-lines reader that yields one typed `Row` at a time from STDIN
or a file path, with schema-on-read column inference and configurable handling
of malformed lines.

## Why

TDD §4.4: input is JSON-lines, one row at a time; malformed lines skip with a
warning by default; `--strict-schema` upgrades to fatal. The streaming
guarantee (O(1) memory for the scan stage) starts here.

## Acceptance criteria

- Reads from either an `io.Reader` (STDIN) or a file path; no buffering of the
  whole input in memory.
- Yields a `Row` abstraction with typed values (string, number, bool, null,
  nested object/array as opaque JSON value).
- Schema-on-read: columns are discovered as rows are seen; column type is the
  first non-null type observed; later type drift handled per `--strict-schema`.
- Malformed JSON line (cannot parse):
  - Default: log `logsql: warn: line N: <reason>` to STDERR and continue.
  - With `--strict-schema`: error and exit with code 3.
- A line counter is tracked for diagnostics in both modes.
- Reader is benchmarkable: a unit benchmark exercises 100k synthetic rows.
- Unit tests cover: valid stream, mixed-type column under default and strict
  modes, malformed line under default and strict modes, empty input,
  trailing-newline edge case.

## Out of scope

- Filtering, projection, aggregation (later tasks).
- Output formatting (task 08).

## Dependencies

01.
