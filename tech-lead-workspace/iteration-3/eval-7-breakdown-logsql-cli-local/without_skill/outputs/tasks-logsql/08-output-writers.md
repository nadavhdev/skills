# 08 — Output writers: CSV and JSON-lines

## Outcome

Two output writers — CSV (default, with header) and JSON-lines (`-o json`) —
that consume rows from the pipeline root and flush as rows arrive.

## Why

TDD §4.4: output is CSV by default with header row, or JSON-lines on
`-o json`. Streaming flush behavior matters for the "pipe to other tools"
use case in §2 and the cold-start NFR in §6.

## Acceptance criteria

- **CSV writer:** writes one header row (unless `--no-headers`); per-row
  flushing; correct escaping per RFC 4180 (quotes, commas, newlines inside
  fields); UTF-8.
- **JSON-lines writer:** one JSON object per line; preserves null vs missing
  semantics; per-row flushing.
- Both writers detect downstream broken pipe (e.g. `| head`) and exit cleanly
  with code 0 — not a panic, not code 3.
- Both writers honor a buffered writer that flushes at row boundaries so the
  first row reaches STDOUT promptly (supports the §6 cold-start NFR).
- Unit tests for CSV: header on/off, escaping cases, empty result, mixed
  types.
- Unit tests for JSON: null/missing distinction, nested objects passed
  through opaquely.
- Integration test: pipe output through `head -n 1` and assert exit code 0.

## Out of scope

- Other output formats (TSV, table, parquet) — explicitly out of v1.

## Dependencies

04 (consumes the pipeline root). Coordinates with 05–07 but does not block on
them.
