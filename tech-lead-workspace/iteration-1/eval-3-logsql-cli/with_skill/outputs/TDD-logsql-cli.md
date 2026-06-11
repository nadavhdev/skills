# TDD: `logsql` — SQL queries over JSON-lines log files

**Author:** Tech Lead (on behalf of Devtools)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-logsql-cli.md` (Dan Park, 2026-05-28)
**Workload category:** CLI utility

---

## 1. Problem & context

On-call engineers debug incidents by downloading `.jsonl(.gz)` log exports
from CloudWatch and grinding through them locally with `jq | grep | awk`
chains and one-off Python. INC-4421 burned 40 minutes finding a correlation
ID that should have taken 2 minutes; the latest DX survey ranks local log
debugging as the #2 worst workflow.

`logsql` is a single-binary CLI that runs SQL-shaped queries (filter,
project, group, sort, limit) against one or more local JSON-lines log
files, streaming records so a 5 GB compressed file fits on a 16 GB laptop.
Consumers are: (a) humans on-call at a terminal, (b) shell pipelines mixing
it with `jq`, `gunzip`, `xargs`, and (c) CI scripts asserting on log
content.

**Why now:** incident retro action item with a deadline, and the
shipping-to-Parquet pipeline doesn't help during a live incident — engineers
need answers from the raw export within the hour.

---

## 2. Scope

**In scope (v1)**

- A single CLI command `logsql` invocable from macOS (Intel + Apple
  Silicon) and Linux x86_64; Windows best-effort.
- Read one or more `.jsonl` / `.jsonl.gz` files (glob expansion via the
  shell; `--` to terminate flags), or `stdin` when no files are given.
- A SQL-shaped query language supporting `SELECT`, `WHERE`, `GROUP BY`,
  `ORDER BY`, `LIMIT`, `COUNT/SUM/AVG/MIN/MAX`, basic comparison and
  boolean operators, and dotted-path column references (`user.id`).
- Output formats: `json` (default, NDJSON), `csv`, `tsv`, `table`
  (human-aligned columns), selected via `--format`.
- Streaming execution where the query allows it; bounded memory for the
  aggregating cases with a documented ceiling.
- Stable, documented exit codes; stderr-only diagnostics; no interactive
  prompts in non-TTY mode.

**Out of scope (explicit, with reason)**

- Joins across files — PRD pushes to v2; combinatorially explodes the
  parser, the planner, and the memory story.
- Remote / S3 / Parquet access — a separate tool exists (`s3sql`,
  per PRD); we will point users at it in `--help` epilog.
- Live tailing of running services — different problem (push, not pull;
  unbounded; partial records). Use `cloudwatch tail` for that.
- Daemon / server mode — no shared state; CLI lifetime = invocation.
- Window functions, CTEs, subqueries, `JOIN`, `HAVING`, `DISTINCT`,
  `UNION` — explicitly rejected for v1; see §5.

---

## 3. High-level approach

**One paragraph.** `logsql` is a Go binary built around DuckDB's
`read_json_auto` table function. The user's SQL string is lightly
rewritten by a thin Go layer (resolves the `FROM`-less form, expands
file globs, picks compression handling), then handed to an in-process
DuckDB instance pointed at the input files as a streaming source.
DuckDB does the heavy lifting (parsing, planning, columnar
vectorised execution, schema inference, spill-to-disk for aggregations
that exceed memory). The Go layer owns the CLI surface, output
formatting, exit codes, and stderr diagnostics. Schema is inferred
on read by DuckDB; we expose a `--schema-hint` flag for the
narrow cases where inference is wrong, but we deliberately do **not**
ship a per-project config file in v1 (see §5).

**Diagram (described).**

```
                +-----------------+
   args / SQL ->|  CLI layer (Go) |
                |  - flag parse   |
                |  - glob expand  |
                |  - SQL rewrite  |
                |  - format pick  |
                +--------+--------+
                         |
                         v
                +-----------------+        +------------------+
                | DuckDB in-proc  |<------>| read_json_auto   |
                | (CGO or native) |        | over file(s)/stdin|
                +--------+--------+        +------------------+
                         |
                         v
                +-----------------+
                | Row formatter   |  -> stdout (json/csv/tsv/table)
                | (streaming)     |  -> stderr (warnings, errors, progress on TTY)
                +-----------------+
```

Engines we considered and rejected are in §5.

---

## 4. Detailed design

### 4a. Command surface

Sketched `--help`:

```
logsql — run SQL over JSON-lines log files

Usage:
  logsql [flags] <SQL> [FILE ...]
  cat file.jsonl | logsql [flags] <SQL>

Arguments:
  SQL           A SELECT statement. The FROM clause is optional and
                defaults to the files given on the command line (or
                stdin). Inside SQL, refer to the input as `logs`.
  FILE          One or more .jsonl or .jsonl.gz files. May be '-' for
                stdin. If omitted, reads stdin.

Flags:
  -f, --format <json|csv|tsv|table>   Output format (default: json
                                       when stdout is not a TTY, table
                                       when it is).
      --no-header                      CSV/TSV: omit header row.
      --schema-hint <col:type,...>     Force types for inference
                                       (e.g. user_id:VARCHAR,
                                       latency_ms:DOUBLE). Repeatable.
      --max-memory <size>              Memory ceiling for the engine
                                       (default 4G). Triggers spill
                                       to a temp dir when exceeded.
      --temp-dir <path>                Spill directory (default
                                       $TMPDIR/logsql-<pid>).
      --explain                        Print the physical plan to
                                       stderr and exit 0.
      --strict-schema                  Fail (exit 4) if a row has a
                                       field whose type conflicts
                                       with inference; default is
                                       to coerce to VARCHAR + warn.
  -t, --timeout <duration>             Abort the query after this
                                       wall-clock time (e.g. 30s).
  -q, --quiet                          Suppress progress on stderr.
  -v, --verbose                        Verbose diagnostics on stderr.
      --version                        Print version and exit.
  -h, --help                           This help.

Exit codes:
  0  success, results returned (possibly zero rows)
  1  generic / unexpected failure
  2  usage / argument error
  3  query parse or planning error
  4  data error (malformed JSON, schema conflict in --strict-schema)
  5  timeout (--timeout exceeded)

For S3/Parquet querying, see `s3sql`.
```

**Input sources.** Files are positional; `-` is stdin; if no positional
file is given and stdin is a pipe, stdin is the implicit input; if no
file is given **and** stdin is a TTY, exit 2 with
`logsql: no input — give a file or pipe data`. Globbing is
shell-driven (we do not re-implement it); inside the SQL string the
user refers to the data set as `logs`. Implicit form
(`logsql 'SELECT path WHERE status >= 500' a.jsonl b.jsonl.gz`) is
rewritten to `SELECT path FROM logs WHERE status >= 500` by the CLI
layer; explicit `FROM logs` is also accepted.

**Output destinations.** Stdout only. We deliberately do not ship
`--output <file>` in v1 — `>` is what shells are for. Diagnostics,
warnings, progress, and `--explain` output go to **stderr**, never
stdout.

**Flag collisions.** None with Go stdlib `flag` or `cobra`. The
`--quiet`/`--verbose` pair is standard; `-f`/`--format` does not
collide with any common Unix idiom for this kind of tool.

### 4b. Exit codes

| Code | Meaning                                                         | Scripts may rely on it? |
|------|-----------------------------------------------------------------|--------------------------|
| 0    | Query ran to completion. **Zero matching rows is still 0.**     | Yes                      |
| 1    | Unexpected error (panic, I/O failure mid-stream, etc.)          | Yes (treat as fatal)     |
| 2    | Usage / argument error (bad flag, no input on TTY)              | Yes                      |
| 3    | SQL parse or planning error (unknown function, bad syntax)      | Yes                      |
| 4    | Data error: malformed JSON line, or schema conflict under `--strict-schema` | Yes        |
| 5    | `--timeout` exceeded                                            | Yes                      |

**Design stance.** We do *not* overload exit code with "no rows found"
(unlike `grep`). For a tool whose primary use is investigation, "the
query ran, the answer is empty" is success, not failure. If a caller
wants to assert non-empty, they can pipe through `[ -s ]` or check the
JSON output. We will document this prominently because it differs from
grep/jq intuition.

Exit codes are stable across minor versions; new codes will be added
above 5 and called out in `CHANGELOG.md`.

### 4c. STDIN / pipe behavior

- `logsql 'SELECT ...' file.jsonl` and
  `cat file.jsonl | logsql 'SELECT ...'` produce identical output.
- `gunzip -c huge.jsonl.gz | logsql 'SELECT ...'` works; we sniff for
  gzip magic on stdin and decompress transparently. Same on file
  inputs (extension `.gz` *and* magic bytes; the latter wins if they
  disagree, with a stderr warning).
- Interactive TTY with no input → exit 2 with a help hint. We do
  not block on stdin in that case.
- Mixed stdin + files: `logsql 'SELECT ...' - a.jsonl` concatenates
  stdin into the input set, in argv order. Documented; no surprise
  ordering.
- We do **not** print a progress bar when stdout is piped or stderr
  is not a TTY. Progress is opt-out via `-q`.

### 4d. Memory & streaming

**The single biggest design choice — taken explicitly.**

DuckDB's `read_json_auto` is a streaming scan; non-aggregating queries
(`SELECT ... WHERE ... LIMIT`) hold only the current batch
(default 2048 rows) plus the output buffer. A 5 GB compressed input
flows row-by-row through the scan and projection operators and never
needs to be materialised.

Aggregating queries (`GROUP BY`, `ORDER BY` without `LIMIT`) require
state proportional to the number of distinct groups. DuckDB's hash
aggregate and external sort **spill to disk** when memory exceeds the
configured limit. We expose this via `--max-memory` (default **4 GB**;
DuckDB default is ~80% of system RAM which is too aggressive for a
laptop also running an IDE and a browser) and `--temp-dir` (default
`$TMPDIR/logsql-<pid>`, cleaned up on exit).

**Concrete bounds we'll commit to in the design and in `--help` epilog:**

- Non-aggregating queries: ≤ 512 MB resident, independent of input
  size, on the PRD's reference 16 GB MBP.
- Aggregating queries: ≤ `--max-memory` resident (default 4 GB); spills
  to disk beyond. Aggregations with O(N) distinct groups (e.g.
  `GROUP BY request_id` with millions of unique IDs) will spill.
- We document that `ORDER BY` without `LIMIT` over a 5 GB input may
  spill tens of GB to disk; users should add `LIMIT` (we'll
  optionally warn on stderr).

**N+1 / fan-out hazards.** None. There is one scan per input file,
read sequentially; DuckDB parallelises across files when given more
than one.

### 4e. Schema-on-read vs config — the engineering fight, settled

The PRD asks: schema config file, or schema-on-read?

**Decision: schema-on-read by default, with a narrow `--schema-hint`
escape hatch. No `.logsql/config.toml` in v1.**

Reasoning:

- Log records are heterogeneous by nature. Two services in the same
  file may use the same key for different types (`user_id` as int vs
  string). A static schema file lies about that and forces users to
  maintain it.
- DuckDB's `read_json_auto` already does the right thing 90% of the
  time: it samples the first N lines (configurable; default 20480),
  infers types, and treats type-mixing columns as `VARCHAR`. This is
  the right default for on-call: never crash on a weird row.
- For the 10% where inference picks the wrong type (commonly:
  `timestamp` inferred as `VARCHAR` because some rows have a `Z`
  suffix and others don't), `--schema-hint user_id:VARCHAR,
  ts:TIMESTAMP` overrides for that single invocation. Repeatable on
  the command line. No file to maintain.
- A config file makes "the same query gives different answers in two
  shells" possible (one shell's CWD has the config, the other
  doesn't). For an on-call tool, that's a footgun.
- We will revisit a config file in v2 **if and only if** users report
  that `--schema-hint` is too verbose in practice. Until then, YAGNI.

We will document the type-inference rules and the
sample-window size in `--help` and in the man page so users can
predict behaviour.

### 4f. SQL parser strictness — the other engineering fight, settled

The PRD asks: ANSI subset, or "looks like SQL" forgiving parser?

**Decision: hand DuckDB the SQL, with a thin pre-rewrite for the
`FROM`-less convenience form. Strict by DuckDB's standards, not by
ANSI's; no forgiving mode.**

Reasoning:

- Writing a SQL parser is a multi-quarter trap. DuckDB's parser is
  production-grade, gives precise error positions, and supports
  exactly the operators the PRD asks for (filter, project, group,
  count, sort, limit) plus dotted-path access via DuckDB's struct
  syntax.
- "Forgiving" parsers (treat `=` as `==`, accept missing commas,
  guess intent) sound friendly and are a constant source of bug
  reports. The cost of typing `=` instead of `==` once is ten
  seconds; the cost of a parser that silently corrects an ambiguous
  query is hours of incident-time confusion.
- We *do* allow one ergonomic shortcut: omitting `FROM logs`. The CLI
  rewrites a bare `SELECT ... [WHERE ...] [GROUP BY ...]` into
  `SELECT ... FROM logs [WHERE ...] [GROUP BY ...]`. This is a pure
  string prepend after the projection list; we do not parse the SQL
  ourselves. If the user supplies their own `FROM`, we leave it alone.
- Diagnostics: when DuckDB returns a parse error, we surface the
  position and the offending token verbatim on stderr. Exit 3.

### 4g. Error reporting

- All diagnostics go to stderr, prefixed with `logsql: `.
- Format for human errors:

  ```
  logsql: parse error at position 27: unexpected token 'GROUP'
    SELECT user_id, COUNT(*) GROUP BY user_id
                             ^
  ```

- Malformed JSON (a row that isn't valid JSON, or has an unterminated
  string) under default settings: skip the line, increment a counter,
  emit one summary warning to stderr at the end:
  `logsql: skipped 12 malformed lines (use --strict-schema to fail fast)`.
  Under `--strict-schema`: exit 4 on the first malformed line, with
  file + line number.
- We do not ship `--json-errors` in v1. Stderr is for humans; if a CI
  script needs to assert on errors, it should rely on the exit code
  and `grep` stderr.

### 4h. External dependencies and what happens if each is down

The CLI has **no network and no external service dependencies at
runtime.** This is a design property the PRD explicitly requires.

Build-time dependencies that matter:

| Dependency           | What it gives us                          | If it's gone / breaks                                  |
|----------------------|--------------------------------------------|--------------------------------------------------------|
| DuckDB (vendored)    | SQL engine, JSON reader, gzip, spill      | Show-stopper; pinned version, vendored statically.     |
| Go stdlib            | CLI, file I/O, gzip fallback              | n/a — stdlib.                                          |
| `cobra` or `flag`    | Flag parsing                              | `flag` is stdlib; cobra is replaceable.                |

Runtime: filesystem (for input, for spill). If `--temp-dir` is unwritable
or full, the engine fails the query and we exit 1 with a clear message.

---

## 5. Alternatives considered

### Language: Go (chosen) vs Python vs Rust

**Chosen: Go.**

- PRD explicitly says devtools team is Go and would like a single
  static binary. Python ships as a `pip` package; Mac engineers
  routinely have broken python toolchains and `brew install logsql`
  beats `pip install logsql` for the "first 60 seconds" goal.
- Go gives us a single static binary cross-compiled to darwin/amd64,
  darwin/arm64, linux/amd64 from one CI matrix. Python wheels would
  need cibuildwheel + manylinux + a separate macOS universal2 build,
  and DuckDB's Python wheel is ~50 MB on its own.
- Python was considered because of the internal `devtools` umbrella
  and faster initial iteration. Rejected: the cost paid is at every
  user install forever, not just at our dev time. The DuckDB Python
  bindings would also pull in the same C++ library — we'd be paying
  for two languages.
- **Rust** considered: would give us the same single-binary story and
  a first-class DuckDB binding via `duckdb-rs`. Rejected: Devtools
  team is Go, not Rust; the long-term maintainer cost of Rust on a
  Go team is real, and the user-visible win over Go is negligible for
  this workload (we are I/O bound on JSON parsing inside DuckDB
  anyway).

**Cost we accept by choosing Go:** DuckDB's Go binding is via CGO,
which complicates cross-compilation (we need a darwin and a linux
build host; we can't cross from linux/amd64 to darwin/arm64 with just
the Go toolchain). Mitigation: GitHub Actions matrix with native
runners per OS/arch, which we already use.

### Engine: DuckDB (chosen) vs SQLite vs custom

**Chosen: DuckDB in-process.**

- DuckDB has `read_json_auto`, columnar execution, spill-to-disk for
  aggregations, and a permissive (MIT) license. It is the right tool
  for "query a JSON file, possibly large, possibly with aggregation"
  — that is literally a documented DuckDB use case.
- **SQLite** considered: ubiquitous, smaller binary footprint (~1 MB
  vs DuckDB's ~20 MB). Rejected: row-oriented, no native JSON
  ingestion (we'd hand-roll the ETL), no spill-to-disk for hash
  aggregates beyond memory, and dramatically slower on the
  `GROUP BY ... ORDER BY ... LIMIT` shape that the PRD user stories
  centre on.
- **Custom engine** considered: write our own scan + filter + hash
  aggregate in Go. Rejected: this is 6 person-months to do badly and
  18 to do well. The PRD wants a 60-second-install tool, not a
  career.
- **Apache DataFusion** (Rust): excellent engine, but committing the
  Devtools team to a Rust transitive dependency for the sake of
  avoiding CGO is a bad trade.

**Cost we accept by choosing DuckDB:** ~20 MB binary (acceptable for
a devtool), CGO build complexity (already addressed above), and we
inherit DuckDB's SQL dialect (the parser is non-ANSI in places, but
documented and stable).

### Schema strategy: schema-on-read (chosen) vs config file vs strict schema

Covered inline in §4e. Short version: inference + per-invocation
hints, no config file, no required schema declaration. Reopen in v2
if `--schema-hint` proves too verbose in real use.

### Parser strictness: DuckDB strict (chosen) vs lenient

Covered inline in §4f. Short version: defer to DuckDB; the only
sugar we add is omitting `FROM logs`.

### Default output: NDJSON when piped, table when TTY (chosen) vs always one

- Always NDJSON: would force humans to pipe through `jq` to read
  anything. Bad on-call ergonomics.
- Always table: breaks pipelines silently.
- TTY-detection switch: standard Unix idiom (`ls`, `git`, `kubectl`
  all do it). User can pin with `--format`.

### Do-nothing

Status quo is `jq | grep | awk` chains. The PRD names a specific
incident (INC-4421) where this cost 38 minutes of on-call time. With
~12 incidents/month touching logs and an average of 15 wasted minutes
each, the payback period for a 4-week build is roughly 4 months. Not
doing it is the worst option.

---

## 6. Non-functional requirements

### Performance (target / mechanism / verification)

- **Target:** time-to-first-result < 5 s on a 1 GB gzipped input on
  2024-era M-series MBP for a `SELECT ... WHERE ... LIMIT 100`
  query. End-to-end < 60 s for a 5 GB gzipped input,
  non-aggregating query.
- **Mechanism:** DuckDB streaming scan with vectorised execution
  (default 2048-row chunks); gzip decompressed in the same goroutine
  as the scan to keep the CPU pipeline full.
- **Verification:** a `make bench` target running against a
  checked-in 100 MB synthetic log and a generated 5 GB gzip fixture
  in CI nightly; we fail the CI if p95 wall-clock for the reference
  query regresses > 20% vs the previous nightly.

### Memory (target / mechanism / verification)

- **Target:** ≤ 512 MB RSS for non-aggregating queries regardless of
  input size; ≤ `--max-memory` (default 4 GB) for aggregating, with
  spill beyond.
- **Mechanism:** DuckDB streaming + bounded memory limit; explicit
  `--max-memory` flag; spill directory.
- **Verification:** `make bench` measures peak RSS via `/usr/bin/time
  -l` (macOS) and `/usr/bin/time -v` (Linux), fails CI on regression.

### Maintainability

- **Test strategy:**
  - Unit tests for the Go CLI layer: flag parsing, FROM rewrite,
    glob expansion, format selection, exit-code mapping. ~70%
    coverage target on this layer.
  - Integration tests: a fixture directory of `.jsonl` and
    `.jsonl.gz` files with known contents; table-driven tests run
    the binary as a subprocess (`exec.Command`) and assert stdout,
    stderr, and exit code. This is the contract.
  - Bench tests: see Performance.
  - We do **not** unit-test DuckDB; we trust its own test suite.
- **Documentation:** `man/logsql.1`, `--help` (autogenerated),
  `README.md` with a 10-line quickstart, and a longer
  `docs/cookbook.md` with the recipes from the PRD's user stories.
- **Ownership:** Devtools team. Bus factor target: 3 (two engineers
  + the team lead familiar enough to ship a patch release).

### Extensibility

- **Variation points:**
  - Output formats: `--format` dispatches through a small
    `Formatter` interface; adding `--format yaml` is a single file.
  - Input formats: today only `.jsonl(.gz)`; the input-resolver
    behind the scenes is shaped to accept `.ndjson`, `.json` (array),
    `.csv` later without restructuring.
  - Schema hints repeatable on the CLI — no syntax change needed
    when more types are supported.
- **Versioning:** semver. Exit codes and `--format json` output are
  the public contract. We will not change either in a minor release;
  breaking changes go in a major release with a deprecation period
  of ≥ 1 minor release, advertised via stderr warning when the old
  form is used.

### Security

- **Threat model:** local-only tool reading user-owned files. No
  network. No privilege escalation. No secrets handled.
- **Input validation:** DuckDB's JSON reader handles malformed input
  safely; the SQL string is passed to DuckDB's parser, not
  interpolated into a shell.
- **Supply chain:** vendored DuckDB pinned to an exact version;
  release binaries signed via Sigstore cosign; SBOM (CycloneDX)
  published with each release. We add a `--version` that also prints
  the embedded DuckDB version and the build commit.
- **What we explicitly do not need:** authn / authz (no callers),
  TLS (no network), secrets management (none handled).

### Cost

- **Build/CI:** GitHub Actions across 3 OS/arch combinations,
  estimated < $50/month at expected commit volume. Within team
  budget; no separate alert.
- **Per-invocation:** local CPU and disk only; cost is the user's
  laptop battery. Not budgeted.

### NFRs explicitly omitted

- **Availability SLO:** N/A — it's a CLI; there is no service to be
  up.
- **Disaster recovery:** N/A — no persistent state. Re-downloading
  the binary is the recovery story.
- **Multi-region:** N/A — local tool.
- **Compliance / data residency:** N/A — data never leaves the
  user's machine. Note: log files themselves may contain PII; that's
  governed by the logging pipeline, not by `logsql`. Users querying
  PII-containing logs locally are subject to existing internal
  policy. We will not write logs ourselves except via stderr (which
  contains diagnostics, never record content).
- **Observability infrastructure:** N/A — see §9 for the CLI
  equivalent (exit codes & diagnostics).

---

## 7. Risks & failure modes

| Risk                                                           | Likelihood | Blast radius                                          | Mitigation                                                                                | Recovery                                                |
|----------------------------------------------------------------|------------|--------------------------------------------------------|-------------------------------------------------------------------------------------------|---------------------------------------------------------|
| DuckDB JSON inference picks wrong type for a field             | High       | Wrong query results, hard to spot                      | `--schema-hint`; document the inference sample window; warn when a column is inferred as VARCHAR after mixed-type lines | Re-run with `--schema-hint`                             |
| OOM on a pathological `GROUP BY` (millions of unique keys)     | Medium     | Single invocation killed                              | `--max-memory` default 4 GB; DuckDB spills to disk; runbook entry showing the symptom    | Add `LIMIT`, raise `--max-memory`, or filter narrower   |
| Spill dir fills up the laptop's disk                           | Medium     | User's machine, not prod                              | Spill dir defaults to `$TMPDIR`, cleaned on exit (incl. SIGINT); document via epilog      | `rm -rf $TMPDIR/logsql-*`                               |
| Malformed JSON line in middle of a 5 GB file                   | High       | Default: warn + skip; `--strict-schema`: exit 4       | Both modes covered; clear stderr summary at end                                          | `--strict-schema` for CI; default for interactive       |
| CGO build breaks on a contributor's machine                    | Medium     | Devtools team velocity                                | Devcontainer + documented toolchain; `make build` in CI is the source of truth           | Fall back to prebuilt release binaries during local dev |
| DuckDB upstream breaking change in `read_json_auto` schema     | Low        | Behavior change between releases                      | Pinned DuckDB version; integration tests run against the pinned binary in CI             | Hold the pin until tests pass on the new version        |
| User pastes a SQL string with shell metacharacters             | Low        | Surprise quoting on the CLI                            | Document quoting in the cookbook; `--help` shows single-quoted example                   | n/a                                                     |
| User runs logsql against a file containing customer PII        | High       | Data exposure within the user's laptop                 | Out of scope for this tool — covered by data-handling policy and laptop encryption       | n/a                                                     |
| Windows build is broken on a release                           | Medium     | Best-effort platform per PRD                          | Windows job is non-blocking in CI; release notes call out platform status                | Document "use WSL on Windows" if needed                 |
| Binary size creeps above 50 MB (Homebrew formula churn)        | Low        | Annoying to install                                    | CI fails the release build if binary > 30 MB                                              | Strip symbols; investigate DuckDB build flags           |

---

## 8. Rollout & migration

There is no production rollout — it's a CLI installed by humans. Phasing
is therefore about *distribution* and *adoption*.

**Phase 0 — internal alpha (week 0-1).** Build from source only.
Devtools team dogfoods. No formula, no pip package. Goal: shake out
the obvious bugs and the `--help` ergonomics.

**Phase 1 — devtools dogfood (week 1-3).** Tap-published Homebrew
formula and a GitHub Release. Announce in `#devtools` and
`#oncall`. Collect feedback. Flag for v1.1 issues we will not block
GA on.

**Phase 2 — GA (week 4).** Announce company-wide. Add a section to
the on-call runbook with the top 5 recipes from the cookbook.

**Backwards compatibility.** N/A for the v1 release. The contract for
future versions is in §6 (Extensibility / Versioning).

**Rollback.** `brew uninstall logsql` or `pip uninstall logsql`. We
will keep the previous release binary downloadable from GitHub
Releases for at least 6 months. Because the tool has no persistent
state, "rollback" really is "use the older binary".

**Feature flags.** None; this is a CLI. New behaviour ships behind a
new flag (default off for behaviour changes; default on for
new features that don't change existing behaviour).

---

## 9. Exit codes & diagnostics (CLI equivalent of observability)

The PRD-relevant observability surface is what the user sees when
something goes wrong. We do **not** ship metrics or telemetry; the
PRD does not require it and the tool runs on engineers' laptops
where telemetry would be ethically and politically expensive.

- **Exit codes:** documented in §4b. Stable across minor releases.
- **Stderr diagnostics:**
  - Errors prefixed with `logsql:`. Always show file + position
    when meaningful.
  - Warnings counted, not printed per-occurrence, then summarised at
    the end:
    `logsql: 12 malformed lines skipped; pass --strict-schema to fail fast.`
  - `--verbose` adds: schema inference summary, row counts per file,
    spill activity (`logsql: spilled 1.2 GB to /tmp/logsql-1234/`).
- **`--explain`:** dumps DuckDB's physical plan to stderr and exits
  0. For debugging slow queries; matches `EXPLAIN` semantics that
  power users already know.
- **`--version`:** prints `logsql/<semver> duckdb/<version>
  commit/<sha> built/<date>`. Required for bug reports.
- **TTY-aware progress:** a single-line spinner with rows-scanned and
  bytes-scanned counters, on stderr, only if stderr is a TTY and
  `--quiet` is not set. Never on stdout.
- **No telemetry phone-home.** Stated explicitly so it doesn't get
  added "for product metrics" later without a separate discussion.
- **Crash diagnostics:** on panic, write a Go stack trace to stderr
  and exit 1. We do not write crash dump files.
- **Runbook (lightweight, since this is a CLI):**
  - "It says `parse error at position N`" → §4f.
  - "It OOM'd on a `GROUP BY`" → raise `--max-memory` or add
    `LIMIT`; §4d.
  - "It's slow on a big file" → run with `--explain`; check if a
    filter could be pushed down (often: cast the JSON field
    earlier).
  - "It said skipped N malformed lines" → run with
    `--strict-schema` to find them; §4g.

---

## 10. Open questions

Numbered, with options and a tentative recommendation where I have
one. The **flagged** ones are decision-flipping and should be
resolved before we commit code.

1. **(Triage / flagged)** Confirm Go is acceptable over Python for
   the production tool. The PRD says either is OK but the team has
   to live with the choice. **Recommendation: Go**, per §5. *If* the
   team picks Python, the design changes meaningfully: distribution
   via pip + a separate manylinux wheel pipeline, DuckDB-Python
   binding, and a packaged entry-point script. Most of this TDD
   still applies, but §5 and §8 need rewriting.

2. **(Triage / flagged)** Is the 5 GB compressed / 16 GB RAM target
   the actual upper bound, or do we need to handle 50 GB
   compressed someday? If the latter, we should validate spill
   behaviour against that size in CI before GA. **Recommendation:**
   commit only to the PRD number for v1; add a non-blocking CI job
   that exercises a 20 GB synthetic input quarterly.

3. **(Triage / flagged)** Windows: best-effort, or supported? PRD
   says best-effort. Devtools survey would clarify. **Recommendation:**
   keep best-effort; ship a Windows binary but mark it as community-
   supported in the README. Revisit if > 10% of users report it
   matters.

4. Should `--schema-hint` accept a file as an alternative to a comma
   list, for when there are 30+ columns? Probably yes
   (`--schema-hint @path.txt`); cheap to add; defer to v1.1 unless
   real usage shows it's painful.

5. Output format for `table` mode: how do we handle very wide values
   (e.g. a full stack trace in one field)? Options: truncate
   (with ellipsis), wrap, or do nothing (let the terminal wrap).
   **Recommendation:** truncate to `--max-col-width` (default 80,
   `0` disables); show the full value in `--format json`.

6. Should `--explain` also print estimated row counts, or just the
   physical plan? DuckDB supports both. **Recommendation:** just the
   plan in v1; add `--explain-analyze` later if anyone asks.

7. Glob expansion when the SQL string and the shell disagree (e.g.
   `logsql 'SELECT...' *.jsonl.gz` on a shell with `nullglob` off
   and no matches). The shell will pass the literal `*.jsonl.gz`,
   which DuckDB will see as a missing file. **Recommendation:** in
   the CLI layer, if a positional argument contains glob
   metacharacters and matches zero files, exit 2 with a helpful
   message rather than passing through to DuckDB.

8. Should we support reading a directory as input
   (`logsql 'SELECT...' ./logs/`)? Useful but adds non-trivial
   semantics (recurse? filter by extension?). **Recommendation:**
   no in v1; tell users to glob (`./logs/*.jsonl.gz`). Revisit in
   v2.

9. Should we publish a `logsql-action` for GitHub Actions so CI
   scripts can use it without a `brew install` step? Cheap;
   probably yes in v1.1.

10. Telemetry: we explicitly do not phone home. Is there a softer
    form (opt-in `logsql --report-bug` that bundles the SQL string
    and stderr) that the team is OK shipping? **Recommendation:**
    yes for v1.1, opt-in only, prints what it would send before
    sending.
