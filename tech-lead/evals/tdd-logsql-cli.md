# TDD: logsql — SQL over log files (CLI)

**Author:** Tech lead (drafted on behalf of the platform team)
**Status:** Draft
**Date:** 2026-06-08
**Related PRD:** `evals/prd-logsql-cli.md`

## 1. Problem & context

Engineers debugging incidents currently grep / awk JSON-lines log files
piped from `kubectl logs`. They want SQL-like queries over those logs
without leaving the terminal. `logsql` is a single-binary CLI tool that
reads JSON-lines from STDIN (or a file path), accepts a SQL-subset
query, and writes results to STDOUT. Target: oncall flow during an
incident — so startup and first-row latency matter.

## 2. Scope

**In scope:** SELECT / WHERE / GROUP BY / ORDER BY / LIMIT over a single
input stream of JSON-lines; column inference (schema-on-read); a small
set of built-in aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`,
`COUNT(DISTINCT)`); CSV and JSON output; piping to other tools.

**Out of scope:** JOINs (one stream in, deliberate); subqueries; window
functions; persistent indexes; remote sources; auth; a daemon mode.

## 3. High-level approach

A Go binary (statically linked, single artifact) using a hand-written
recursive-descent parser for the SQL subset. The executor streams input
through a pipeline of operators: scan → filter (WHERE) → project
(SELECT) → aggregate (GROUP BY) → sort (ORDER BY) → limit. Streaming
operators (filter, project, limit) pass rows through one at a time; the
two blocking operators (aggregate, sort) accumulate in memory with a
**named, enforced memory budget** (default 512 MB, configurable via
`--max-memory`).

**Language: Go.** Static binary is a hard requirement for distribution
on engineer laptops without a runtime install. Python would otherwise
be a credible choice but loses on cold-start latency and packaging.

**Parser strictness: strict.** Unknown SQL constructs are a parse error
with a column-pointing message, not a permissive best-effort. Reasoning:
debugging an incident with subtly-wrong query semantics is worse than
having to fix the query.

## 4. Detailed design

### 4.1 Command surface

```
logsql [-f FILE | <STDIN>] [-o csv|json] [--max-memory SIZE]
       [--no-headers] [--strict-schema] 'SELECT ...'
```

- No input file → read STDIN. Both → error.
- Default output: CSV with a header row.
- `--strict-schema`: error on the first row where a referenced column
  is missing or the wrong type; default is to coerce / null.

### 4.2 Parser

Recursive-descent, AST output. Error messages point to the column in the
query string. No external SQL parser dependency.

### 4.3 Executor

Pipeline of operators implementing a `Next() (Row, error)` interface.
Streaming operators (scan, filter, project, limit) hold O(1) memory.
Blocking operators (aggregate, sort) accumulate; when memory budget is
exceeded, return a typed error with the column and operator that blew
the budget (no silent spill-to-disk in v1).

### 4.4 IO

- Input: JSON-lines reader, one row at a time. Malformed lines: skip
  with a warning to STDERR by default; `--strict-schema` upgrades to
  fatal.
- Output: CSV writer (default) or JSON-lines writer (`-o json`). Writes
  flush as rows arrive for streaming operators; for blocking operators,
  flush starts once the operator completes.

### 4.5 Exit codes

- `0` — query ran, output produced (even if zero rows).
- `1` — query ran, **zero rows matched** (deliberate: useful in pipelines
  to distinguish "no match" from "error"). New convention; documented.
- `2` — parse error (argparse-style).
- `3` — runtime error (memory budget, IO error, type mismatch under
  `--strict-schema`).

## 5. Alternatives considered

- **`jq` + shell.** Rejected: composable but the SQL-shaped queries we
  want require non-trivial `jq` chains that nobody writes during an
  incident.
- **`duckdb` invoked over JSON.** Rejected: too heavy a dependency for
  a `kubectl logs | ...` use case; cold start is 100s of ms.
- **Embed an existing SQL parser library.** Rejected: every option pulls
  in a large surface (window functions, JOINs, dialects) we don't want
  to inherit or document.
- **Python with a packaging story (pipx, uv).** Rejected on the static-
  binary requirement; cold start also worse.

## 6. Non-functional requirements

- **Cold-start latency.** First row to STDOUT < 100ms for a simple
  `SELECT * WHERE ... LIMIT 10`. *Verified:* benchmark in CI.
- **Throughput.** 1 GB / 60s for a streaming `SELECT WHERE LIMIT` query
  on a laptop. *Verified:* benchmark with synthetic 1GB JSON-lines file.
- **Memory.** Hard budget enforced; default 512 MB.
- **Availability.** N/A — local CLI.
- **Durability.** N/A — no persistent state.
- **Security.** N/A — reads only what the user can read.
- **Cost.** N/A.
- **Compliance.** N/A.
- **DR.** N/A.

## 7. Risks & failure modes

- **Memory blow-up on large GROUP BY.** Surfaced as runtime error with
  the offending operator named; user can re-run with `--max-memory` or
  add a `LIMIT`.
- **Schema drift mid-stream.** Default behavior (coerce/null) keeps the
  pipeline going; `--strict-schema` fails fast for users who care.
- **Subtly wrong SQL semantics.** Mitigated by strict parser and a
  documented subset; anything not in the documented subset is a parse
  error, not silent.

## 8. Rollout & migration

- Distribute as a Homebrew formula and a direct binary download from
  the internal package repo.
- v0.1 to platform-team volunteers for one week.
- v0.2 publishes broadly with the documented SQL subset frozen.
- Backwards compat: N/A — v1 release.
- Rollback: users `brew uninstall logsql` or use an older binary.

## 9. Exit codes & diagnostics

(Replaces "observability" for this CLI workload.)

- All errors go to STDERR with `logsql: <category>: <message>` prefix.
- `--explain` prints the parsed AST and the operator pipeline for the
  given query, runs nothing.
- Crash reports: panic handler prints a stable issue-report block
  (version, args, stack frame) to STDERR.

## 10. Open questions

1. **Exact SQL subset.** The parser strictness decision means the
   documented subset is the contract. We need a final review of
   which functions / operators are in: e.g. string functions
   (`LOWER`, `LIKE`), date functions, `CASE`. Recommend: ship v1
   with `LIKE`, `LOWER`, `UPPER`, `COALESCE`, and `CASE`; defer
   date functions to v1.1. Needs platform-team sign-off.
