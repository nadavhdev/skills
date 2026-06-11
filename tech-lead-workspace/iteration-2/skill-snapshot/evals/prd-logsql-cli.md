# PRD: `logsql` — SQL queries over JSON-lines log files

**Author:** Dan Park (Developer Productivity)
**Date:** 2026-05-28
**Status:** Eng design pending

## Background

Our backend services emit structured JSON logs (one record per line) and we
ship them to S3 in hourly Parquet files via a separate pipeline. For
real-time debugging, on-call engineers download the raw `.jsonl.gz` files
from CloudWatch Logs Insights export and dig through them locally.

Today people use a mix of `jq`, `grep`, `awk`, and ad-hoc Python scripts.
This is slow and error-prone — last incident retro (INC-4421) called out
that on-call spent 40 minutes trying to find a correlation ID across 12
log files when it should have taken 2 minutes. The DX survey put "log
debugging on local machines" as the #2 most-painful workflow.

We want a small CLI, `logsql`, that lets engineers run SQL-like queries
against one or more local `.jsonl` / `.jsonl.gz` log files.

## Goals

- Single static binary or single-file install — engineers should be able
  to `brew install logsql` or `pip install logsql` and be productive in
  under 60 seconds.
- Run a SQL-like query over one or many log files, including `.gz`.
- Support common operations: filter, project columns, group by, count,
  sort, limit. Joins are out of scope for v1.
- Stream the output — handle a 5 GB compressed log file without OOMing on
  a laptop with 16 GB RAM.
- Output as JSON (default), CSV, TSV, or human-friendly aligned columns.
- Read STDIN when no files given (so it composes with other CLIs).
- Useful in shell pipelines and CI scripts (exit codes, machine-readable
  output, no interactive prompts in non-tty mode).

## Non-goals

- Querying remote S3 / Parquet files directly (a separate tool exists for
  that — point users at it).
- Real-time tailing of live log streams.
- Joining across multiple files (v2).
- A daemon / server mode.

## User stories

1. As on-call, I can run `logsql 'SELECT * WHERE error_code = "ECONN" AND
   service = "billing"' service-*.jsonl.gz` and get matching records.
2. As an engineer, I can run `logsql 'SELECT COUNT(*) GROUP BY user_id
   ORDER BY 1 DESC LIMIT 10' app.jsonl` to find noisy users.
3. As a CI script, I can run `logsql --json '...query...' file.jsonl |
   jq '...'` and have machine-readable output.
4. As a user piping in, `gunzip -c huge.jsonl.gz | logsql 'SELECT path,
   COUNT(*) GROUP BY path ORDER BY 2 DESC LIMIT 20'` should work.

## Constraints

- Devtools team is Go (and we'd like a single static binary), but we
  also have an internal Python platform (`devtools` umbrella) — the
  team is open to either.
- Must work on macOS (Intel + Apple Silicon) and Linux (x86_64). Windows
  is best-effort.
- No external service dependencies — runs entirely locally.
- Must work without an internet connection.
- Target time-to-first-result for a 1 GB compressed input < 5 seconds on
  a 2024-era MacBook Pro.

## Open from PM side

- Should we support a `.logsql/config.toml` for per-project schema hints,
  or keep it schema-on-read with auto-inference? (Engineers have opinions
  here.)
- How strict should the SQL parser be — exact ANSI subset, or a
  forgiving "looks like SQL" parser?
