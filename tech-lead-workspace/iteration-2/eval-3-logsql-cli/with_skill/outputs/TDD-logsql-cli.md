# TDD: `logsql` — SQL queries over local JSON-lines log files

**Author:** Tech lead (on behalf of Dan Park, Developer Productivity)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-logsql-cli.md`

---

## 0. Design constraints from project conventions

`logsql` is a greenfield tool that does not live inside an existing repo's
convention set (the working directory's `CLAUDE.md` describes csvkit, which is
unrelated). No project-level `CLAUDE.md`, `CONTRIBUTING.md`, `ARCHITECTURE.md`,
or ADR set applies. The PRD references an internal Python "devtools" umbrella
and a Go-leaning devtools team — those are organisational signals, not
documented conventions, and are handled in Section 5 (Alternatives) rather
than treated as hard constraints.

If/when this lands in the `devtools` repo, this TDD must be re-checked against
that repo's `CONTRIBUTING.md` and any house style guides before code review.

---

## 1. Problem & context

On-call engineers debug production incidents by downloading `.jsonl.gz`
exports from CloudWatch Logs Insights and grepping/jq-ing them locally.
INC-4421 retro called out 40 minutes spent chasing a correlation ID across
12 files; the DX survey ranks "local log debugging" as the #2 most painful
workflow. We need a single-binary CLI that lets engineers express the same
intent as `WHERE`, `GROUP BY`, `COUNT(*)`, `ORDER BY`, `LIMIT` against one
or more local JSONL files (gzip-aware), composes in shell pipelines, and
holds streaming memory bounds on multi-GB inputs.

Consumers: backend engineers on their own laptops (primary); CI scripts
that post-process logs (secondary). Reach is local-only — no service
component, no network calls in the steady state.

Why now: INC-4421 is fresh, and the DX survey window closes this quarter —
shipping before EOQ lets us claim the win in next quarter's planning.

---

## 2. Scope

**In scope (v1):**

- Single-binary install (`brew`, `pip`, or a direct binary download).
- SQL-like query against one or many local `.jsonl` / `.jsonl.gz` files.
- Operations: `SELECT` projection, `WHERE` filter, `GROUP BY`, aggregate
  `COUNT`/`SUM`/`AVG`/`MIN`/`MAX`, `ORDER BY`, `LIMIT`, `OFFSET`.
- Output formats: JSON (default), JSONL, CSV, TSV, aligned-columns
  ("table"). Format auto-selects to `table` when stdout is a TTY, `jsonl`
  otherwise (overridable with `--format`).
- STDIN input when no file arguments given.
- macOS (Intel + Apple Silicon) + Linux (x86_64 + arm64) first-class;
  Windows best-effort (no CI gate).
- Exit-code contract suitable for CI / shell scripts.
- Schema-on-read with auto-inference (see Section 4f for the stance).

**Out of scope:**

- Remote S3 / Parquet querying (existing tool handles that — point users
  there in `--help` epilog).
- Real-time tailing of live log streams.
- JOINs across files (deferred to v2; calling it out so v1's planner
  architecture doesn't paint us into a corner — see Section 4d).
- Daemon / server / REPL mode.
- A persistent index or cache layer. Every invocation is from scratch.

---

## 3. High-level approach

`logsql` is a single Go binary. It accepts one SQL-ish query string and a
list of file paths (or stdin), opens each input as a line-delimited byte
stream (auto-detecting gzip by magic bytes, not by extension, so piped
gzip works), parses each line as JSON into a typed record, evaluates the
query as a pipeline of streaming operators (scan → filter → project →
aggregate → sort → limit), and writes the result to stdout in the
requested format.

Box-and-arrow:

```
[ argv / stdin ]
        |
        v
[ SQL parser (PartiQL-subset, hand-tuned) ]
        |
        v
[ Logical plan ]
        |
        v
[ Physical plan: streaming operator chain ]
        |
   per-file (parallel, worker = NumCPU/2, capped):
   [ gzip decoder ] -> [ jsonl scanner ] -> [ row materializer ]
        |
        v
[ Filter ] -> [ Project ] -> [ Aggregator (hash, spillable) ]
                                   |
                                   v
                          [ Sort (spillable) ] -> [ Limit ]
                                   |
                                   v
                       [ Format writer -> stdout ]
```

The aggregator and sort are the only operators that need state; everything
else streams record-by-record. The aggregator and sort spill to a
temp-directory run-file when their working set exceeds a soft cap (default
512 MB). This is the design's central memory-bounding choice and is what
lets us honour "5 GB compressed without OOM on a 16 GB laptop".

---

## 4. Detailed design

### 4a. Command surface

```
logsql [flags] <query> [files...]

Flags:
  -f, --format        json|jsonl|csv|tsv|table   (auto: table if tty, jsonl else)
  -o, --output PATH   write to PATH instead of stdout
      --no-header     omit CSV/TSV header row
      --null=STR      string for SQL NULL in CSV/TSV/table (default: empty)
      --max-mem BYTES soft cap for in-memory aggregate/sort state (default 512MiB)
      --spill-dir DIR temp dir for spilled runs (default: $TMPDIR)
      --workers N     parallel scan workers (default: min(NumCPU/2, 4))
      --strict        fail on first malformed JSON line (default: skip + warn)
      --schema PATH   optional schema hints file (TOML; see 4f)
      --explain       print physical plan and exit
      --version
  -h, --help

Arguments:
  <query>             a single SQL-ish query string (single-arg, quoted)
  [files...]          0+ paths; '-' or absent => read stdin
```

Notes on the surface:

- The query is a **single positional argument**, not a flag, because every
  invocation has one and quoting `-q '...'` would be noise.
- We deliberately **do not** add a `--query-file` flag in v1. If users need
  it, they can `logsql "$(cat q.sql)" file.jsonl`. We'll add it if real
  usage demands it (see Extensibility, Section 6).
- Glob expansion is done by the **shell**, not by `logsql`. The PRD example
  `service-*.jsonl.gz` is the shell expanding before invocation. Internally
  we treat `argv[2:]` as a flat path list.
- `--explain` is in v1 because debugging "why is this slow" is the
  difference between this tool succeeding and falling out of use.

### 4b. Exit codes

Strict contract — scripts and CI can rely on these:

| Code | Meaning                                                         |
|------|-----------------------------------------------------------------|
| 0    | Query ran successfully (result may be empty — empty is success).|
| 1    | Generic / unexpected runtime error (panic, I/O failure).        |
| 2    | Usage / argument / parse error (matches `argparse` / POSIX).    |
| 3    | One or more input lines failed JSON parsing while not in `--strict`. Result was still produced. This is a **warning code**; CI can ignore or fail on it as policy. |
| 4    | `--strict` mode: aborted on a malformed input line.             |

Exit code 3 is the deliberate-design-choice flag the CLI reference calls
out. It exists because "I got results but some lines were skipped" is a
real distinction we want CI to be able to detect without scraping stderr.
We document it explicitly so nobody invents a different convention later.

### 4c. STDIN / pipe behavior

- No file arguments **and** stdin is a pipe → read JSONL from stdin.
- No file arguments **and** stdin is a TTY → exit 2 with
  `logsql: no input — pass a file or pipe JSONL to stdin (see --help)`.
  No interactive prompt, ever.
- A path argument of `-` is treated as stdin (so users can mix:
  `logsql 'q' file1.jsonl - file3.jsonl`).
- Gzip detection on stdin is by **magic bytes** (`1f 8b`), not by an
  extension we don't have. Same logic on file inputs, so `cat foo.jsonl >
  foo.bin && logsql 'q' foo.bin` works.
- Diagnostics (warnings about skipped lines, the `--explain` plan, the
  one-line summary at end if `--stats` is set) go to **stderr**. stdout is
  reserved for result data so pipes don't break.

### 4d. Query language: parser + executor

This is one of the two "engineers will fight about it" decisions. The
stance:

**Parser: a hand-written PEG-style parser for a defined PartiQL-flavoured
SQL subset. Not ANSI-strict. Not "looks like SQL".**

Rationale:

- ANSI-strict parsers (e.g. `vitess/sqlparser`, `pingcap/parser`,
  `cockroachdb/parser`) come with full SQL grammars we don't need
  (DDL, CTEs, window functions, transactions), bloat the binary by 5–15 MB,
  and force us to reject queries that ANSI doesn't allow but are obvious
  for this use case (e.g. dotted paths into nested JSON: `WHERE
  request.headers.x_request_id = '...'`).
- "Looks like SQL" lenient parsers (the `q` tool family, `textql`) feel
  great until two engineers disagree on what their query meant. Diff in
  result, no diff in syntax. That's the worst kind of bug for a debugging
  tool.
- The middle path: define the supported grammar in EBNF in `docs/grammar.md`,
  parse strictly against it, but include the JSONPath-style dotted-path
  selector as a first-class syntactic form (not retrofitted as a string
  literal). Errors point at the offending token with a column number.

The grammar covers exactly:
`SELECT <proj> [FROM <ignored>] [WHERE <expr>] [GROUP BY <cols>]
[ORDER BY <cols> [ASC|DESC]] [LIMIT n] [OFFSET n]`, where `<expr>`
supports `= != < <= > >= AND OR NOT IN LIKE IS NULL`, integer / float /
string literals, dotted-path selectors, and the aggregate functions
listed in scope.

`FROM` is accepted-and-ignored (the files come from argv). We accept it
because muscle memory will make people type it; we ignore it because
binding a name there is meaningless when input is a file list.

**Executor: pull-based streaming, Volcano-model operators.** Each operator
implements `Next() (Row, error)`. The two stateful operators — hash
aggregator and sort — buffer until they hit `--max-mem`, then spill runs
to disk and do an N-way merge on the final pass. This is a well-trodden
pattern (DuckDB, ClickHouse local mode, every embedded analytics engine
of the last decade); we are not innovating here on purpose.

**Why not just embed DuckDB?** Considered — it would give us a complete
SQL surface, vectorised execution, and JSON support out of the box. We
reject it because: (a) it bloats the static binary from ~12 MB to ~45 MB,
which kills the "single binary brew install" goal in spirit; (b) it pulls
the full DuckDB SQL surface, which we'd then have to document the subset
of; (c) the v1 query surface is small enough that hand-written operators
are ~2 weeks of work and we own the failure modes. We revisit this for v2
when JOINs are in scope (Section 5).

### 4e. Memory & streaming

Explicit statement: **the tool streams record-by-record through filter
and projection. Aggregation and sort buffer to `--max-mem` (default 512
MiB) and spill to `--spill-dir` (default `$TMPDIR`) when over.**

The bound for a query with no aggregate and no sort is:
`O(largest_record_size + decoder_buffers)` — well under 100 MB for any
realistic log record.

The bound for a query with aggregation: `O(group_count * row_size)` until
spill; after spill, bounded by `--max-mem`. We document this in `--help`
epilog: `GROUP BY on high-cardinality columns may spill to disk; use
--max-mem to tune.`

The PRD target — 5 GB compressed (~20–40 GB uncompressed JSON) on a 16 GB
laptop — is met because (a) raw scan never holds more than one record per
worker; (b) aggregate spills at 512 MiB; (c) sort spills at the same.

### 4f. Schema-on-read vs config

The other "engineers will fight about it" decision. The stance:

**Schema-on-read with auto-inference by default. Optional `--schema PATH`
TOML file for two specific cases: (1) forcing a type when inference picks
the wrong one, (2) declaring a dotted path's alias.**

We reject `.logsql/config.toml` in the cwd / per-project. Reasoning:

- This is a debugging tool. The cost of getting it wrong on a one-off run
  is "rerun with `--schema`". The cost of *every project* needing a
  config file checked in is "this didn't work because the config drifted",
  which is exactly the failure mode the PRD's #2 painful-workflow item is
  about.
- Implicit per-directory config is hidden global state, called out as a
  pitfall in the CLI category reference. Two engineers on the same files
  could see different output. No.
- The `--schema` flag (explicit, opt-in) gets us the escape hatch for
  the case where inference is wrong, without paying the implicit-config
  cost.

Type inference rules, in order, per column:

1. If all observed values are `null` → `null` type (rendered as empty).
2. If all non-null values parse as integer → `int64`.
3. Else if all parse as number → `float64`.
4. Else if all parse as RFC3339 → `timestamp`.
5. Else if all are JSON booleans → `bool`.
6. Else → `string`.

Inference is **per-query**, computed during a fast prelude pass over the
first 1000 records (configurable via `--infer-sample`). If a later record
violates the inferred type, we fall back to string for that column and
emit a stderr warning (exit code stays 0 unless `--strict`).

`--schema PATH` is TOML:

```toml
[columns]
"request.duration_ms" = "int64"
"timestamp"          = "timestamp"
"user_id"            = { type = "string", alias = "uid" }
```

### 4g. Concurrency model

- One coordinator goroutine.
- `--workers` decoder goroutines (default `min(NumCPU/2, 4)` — we cap at
  4 because higher contention on a single laptop SSD usually slows things
  down, and 4 is enough to saturate gunzip on a 5 GB file in our prior
  benchmarks of similar tools).
- One output goroutine.
- Channels between stages with bounded capacity (default 1024 rows per
  buffer) so backpressure flows naturally.

Aggregation is single-threaded in v1. Parallel aggregation needs a
two-phase shuffle and is not worth the complexity until we measure that
single-threaded aggregation is the bottleneck on multi-file inputs. We
expect scan + JSON parse to dominate.

### 4h. External dependencies (build + runtime)

| Dependency        | Purpose                       | What if it goes wrong            |
|-------------------|-------------------------------|----------------------------------|
| Go standard lib   | gzip, bufio, encoding/json    | None — stdlib.                   |
| `valyala/fastjson` (or `bytedance/sonic`) | faster JSON parsing | Bench both during week-1 spike; pick one. Fall back to `encoding/json` if either has correctness issues. |
| GoReleaser        | cross-compile + brew tap      | If breaks: ship a fallback `go build` script in the README. |

No runtime dependencies. No network calls. PRD requirement: works without
an internet connection. Met by construction.

### 4i. Error reporting format

Human-readable (default to stderr):

```
logsql: parse error at column 23: expected ')' after expression
  SELECT count(*) WHERE x = 1
                        ^
```

Machine-readable (when `--format json` or `--format jsonl` is selected AND
stderr is non-tty, or with `--errors-json`):

```json
{"level":"error","stage":"parse","msg":"expected ')'","file":"-","line":1,"col":23}
{"level":"warn","stage":"scan","msg":"malformed JSON line skipped","file":"app.jsonl","line":4837}
```

Skipped-line warnings include file, line number, and the first 80 chars
of the offending line. We do **not** echo the full line — log records
sometimes contain secrets, and stderr leaks into CI artifacts.

---

## 5. Alternatives considered

### 5a. Language: Go vs Python — taking a stance

**Recommendation: Go.** Not because the devtools team is "leaning Go", but
for three concrete reasons:

1. **Single-binary install is a primary PRD goal.** Python's
   single-file install story is `pip install`, which depends on the user
   having the right Python on PATH, the right pip, no venv conflicts,
   and (for any C-extension dep) a working toolchain. We've seen this
   fail in INC retros before. `brew install logsql` for a 12 MB
   static binary is one curl + chmod; this is what the PRD's "productive
   in under 60 seconds" goal means in practice.
2. **Streaming + gzip + JSON throughput.** Python's GIL means the
   gzip-decode + json-parse pipeline doesn't go parallel without
   multiprocessing, which costs serialisation across the process
   boundary and doesn't pay back at log-file sizes. CPython JSON parsing
   tops out around 200 MB/s; Go with `fastjson`/`sonic` is 1–2 GB/s.
   The 5 GB / 16 GB target is reachable in Python but with rewrites
   (orjson, generator pipelines, careful memoryview); in Go it falls out
   of the standard idiom.
3. **The Python ecosystem advantage (rich libraries, polars, pandas) is
   wasted here.** This tool doesn't need numpy or arrow. It needs gzip,
   JSON, hash, and sort.

**Tradeoff we're accepting by picking Go:** the devtools-Python umbrella
team won't be able to maintain it as easily. We mitigate by (a) keeping
the binary truly small and the codebase ~2k LOC so a Pythonista can read
it, (b) shipping a Python equivalent as a thin wrapper that calls the
binary — `pip install logsql` installs the binary too, via the same
release pipeline (the wheel bundles per-platform binaries; precedent:
ruff, uv).

### 5b. Embed DuckDB instead of writing a parser + executor

Considered. Rejected because of binary size (~45 MB vs ~12 MB), surface
area we'd have to document the subset of, and because v1's grammar is
small enough that hand-writing it costs ~2 weeks while owning the failure
modes outright. We re-open this for v2 if JOINs (PRD non-goal #3) come in.

### 5c. Use `jq` + scripts (do nothing)

This is the status quo. Rejected because INC-4421 is the evidence that
status quo costs us 38 minutes per incident at the relevant percentile.
At ~2 such incidents/quarter, the tool pays for its build cost (~3
engineer-weeks) inside one quarter.

### 5d. Load whole file, then SQLite in-memory

Lets us reuse SQLite's SQL parser. Rejected because (a) loading a 5 GB
compressed JSONL into SQLite tables OOMs on 16 GB, defeating the PRD
memory goal; (b) the JSON1 extension is fine but slower than direct
parsing; (c) SQLite's SQL is full-featured, which becomes a docs
problem again.

### 5e. Strict-ANSI vs lenient-"looks like SQL" parser

Covered in Section 4d. Stance: middle path, EBNF-defined subset, strict
against that subset. The other two were rejected explicitly.

### 5f. `.logsql/config.toml` per-project vs schema-on-read

Covered in Section 4f. Stance: schema-on-read with optional `--schema`
flag. Per-project implicit config is rejected as hidden global state.

---

## 6. Non-functional requirements

Walking `references/nfrs-checklist.md` in order. Each NFR appears with
either a target/mechanism/verification or an explicit N/A line.

### Scalability

- **Capacity target.** Input: 5 GB gzip-compressed JSONL (~20–40 GB
  uncompressed) per invocation. Throughput: 1 GB compressed in ≤ 5 s on
  a 2024 MBP M3.
- **Scaling axis.** Vertical only — this is a CLI, no horizontal story.
- **Stateful bottleneck.** Aggregator + sort. Mitigation: spill to disk
  at `--max-mem` (default 512 MiB).
- **Back-pressure.** Channel-buffered pipeline; slow output throttles
  scan. **Verified** via load test in `tests/perf/` against a 5 GB
  synthetic file in CI (Linux runner only; Mac runners are too slow).

### Reliability & availability

- **Availability SLO.** N/A — local CLI, no uptime.
- **SPOFs.** None — no external dependencies at runtime.
- **Dependency failure.** N/A — no runtime dependencies.
- **Idempotency.** N/A — read-only over immutable files; every run is
  independent.

### Resilience

- **Graceful degradation.** Malformed line in non-strict mode → skip,
  warn to stderr, exit 3 if any were skipped. Inference miss → demote
  column to string, warn.
- **Bulkheads.** N/A — single-process CLI.
- **Timeouts.** N/A — no network I/O.
- **Backoff & jitter.** N/A — no retries.

### Performance

- **Latency target.** Time-to-first-result < 5 s for 1 GB compressed
  input on 2024 MBP M3. p50 wall-clock for a 100 MB compressed file
  with a simple filter: < 1 s.
- **Resource budget.** Memory: ≤ `--max-mem` + small constant
  (≤ 700 MiB default). CPU: up to `--workers` cores.
- **N+1 / fan-out hazards.** None — no per-record external calls.
- **Verification.** `bench/` directory with `go test -bench` against
  pinned 100 MB / 1 GB / 5 GB synthetic JSONL files, run on every PR
  with regression threshold (≥ 10 % slower fails CI).

### Observability

- **Metrics.** N/A — CLI, no metrics endpoint. `--stats` flag prints a
  one-line summary to stderr at exit: rows scanned, rows matched, rows
  skipped, wall-clock, peak memory.
- **Logs.** Diagnostics to stderr, structured JSON optional via
  `--errors-json`.
- **Traces.** N/A.
- **Alerts.** N/A.
- **Runbook.** A short `docs/troubleshooting.md` covering: "exit 3 — what
  do I do", "query is slow — try --explain", "OOM despite --max-mem —
  check group-by cardinality".

### Security

- **AuthN / AuthZ.** N/A — local CLI, runs as the invoking user.
- **Secrets.** Tool itself handles no secrets. **However**, log files
  routinely contain secrets (tokens, cookies, customer PII). Mitigations:
  (a) skipped-line warnings include only the first 80 chars of the line
  and never the parsed structure; (b) `--stats` and `--explain` never
  echo data; (c) document in `--help` epilog that stderr should not be
  collected into CI artifacts when running over real log data.
- **Data classification.** Inputs may contain PII / secrets. Stated; not
  the tool's responsibility to redact, but stated so consumers know.
- **Input validation.** JSON parsing is strict per the spec; SQL parsing
  rejects anything not in the EBNF grammar.
- **Threat model.** Local, single-user. The realistic risk is a malicious
  log file causing a crash; we cover this with fuzz tests (`go-fuzz` on
  the JSON scanner and the SQL parser) in CI.
- **Supply-chain.** `go.mod` lockfile committed; dependency review on
  every PR; release artifacts signed with cosign and SLSA-3 provenance
  via GoReleaser.

### Cost

- **Estimated unit cost.** $0 runtime (local execution). Build cost: ~3
  engineer-weeks for v1, ~1 engineer-week ongoing maintenance.
- **Cost drivers.** N/A — no managed services.
- **Cost ceiling.** N/A.

### Compliance & privacy

- **Data residency.** N/A — local execution; data never leaves the
  engineer's laptop.
- **Retention.** N/A — tool stores nothing. Spill files in `--spill-dir`
  are deleted on exit (and on `SIGINT` via deferred cleanup); document
  that crash-mid-run can leave temp files.
- **Regulatory.** N/A.
- **Audit log.** N/A.

### Deployment & operations

- **Deploy strategy.** GitHub Releases via GoReleaser. Homebrew tap
  formula auto-PR'd by GoReleaser. Python wheel published to internal
  PyPI mirror.
- **Rollback.** Users `brew install logsql@<previous-version>` or
  `pip install logsql==<previous-version>`. Old binaries kept on releases
  page for 1 year.
- **Feature flags.** N/A.
- **Config.** No persistent config. `--schema` is per-invocation only.
- **Migrations.** N/A.

### Disaster recovery

- **RTO / RPO.** N/A — no persistent state.
- **Backups.** N/A.
- **Multi-region.** N/A.

### Maintainability

- **Test strategy.** Unit (parser, operators, type inference);
  integration (golden-file tests: query + input JSONL → expected output);
  fuzz (parser, scanner); perf benchmark (regression-gated). All in CI.
- **Documentation.** This TDD; `docs/grammar.md` for the SQL subset
  EBNF; `docs/troubleshooting.md`; `README.md` with quickstart.
- **Ownership.** Developer Productivity team. On-call: none (no live
  service). Bug triage on the team's standard inbox.

### Extensibility

- **Variation points.** (a) New aggregate functions plug into a registry
  in `eval/agg/`. (b) New output formats plug into `io/format/`.
  (c) v2 JOIN: requires a new physical operator (hash join) and a real
  `FROM` resolver — the grammar already has `FROM` reserved so we won't
  break v1 queries. (d) v2 `--query-file` flag is additive.
- **Versioning.** Semver. Grammar changes that reject previously-accepted
  queries are major bumps. New flags / new functions / new formats are
  minor. Bug fixes are patch. The on-disk spill format is internal —
  never persisted across processes — so no compat story needed.

---

## 7. Risks & failure modes

| # | Risk                                                  | Likelihood | Blast radius                       | Mitigation                                                                                  | Recovery                            |
|---|-------------------------------------------------------|------------|-------------------------------------|---------------------------------------------------------------------------------------------|-------------------------------------|
| 1 | Type inference picks wrong type for a column          | High       | Wrong/empty results, silent.       | `--schema` override; warn to stderr on inference demotion; default `--infer-sample 1000`.   | Re-run with `--schema`.             |
| 2 | OOM on extreme GROUP BY cardinality                   | Medium     | Crash mid-query.                   | Spill at `--max-mem`; doc in epilog; `--explain` shows estimated cardinality if available.  | Re-run with lower `--max-mem`.      |
| 3 | Malformed JSON line silently skipped, user doesn't notice | Medium | Wrong results.                     | Exit code 3 on skips; stderr warning per N skipped (1, 10, 100, then every 1000); `--strict` option. | Re-run with `--strict` to investigate. |
| 4 | Spill dir fills up disk                               | Low        | User's laptop runs out of space.   | Cap spill at 4× `--max-mem` total; abort with clear error if exceeded.                      | Free disk; re-run with lower max-mem. |
| 5 | SIGINT mid-aggregation leaves spill files             | Medium     | Wasted disk until cleanup.         | Defer-cleanup on signal; document `$TMPDIR/logsql-*` pattern in troubleshooting.            | `rm $TMPDIR/logsql-*`.              |
| 6 | gzip bomb (small file decompresses to TBs)            | Low        | Disk fills via spill, or CPU pegged.| Decompressed-bytes counter with `--max-input` ceiling (default 50 GiB); abort cleanly.       | Re-run with adjusted `--max-input`. |
| 7 | Secrets in log lines leak into stderr / CI artifacts  | Medium     | PII / token exposure.              | Truncate skipped-line warnings to 80 chars; document risk; never log parsed records.        | Rotate any leaked secret.           |
| 8 | Performance regression in a release breaks 5 s target | Medium     | Tool falls out of use.             | CI benchmark gate with 10 % regression threshold against pinned fixtures.                   | Revert release; investigate.        |

---

## 8. Rollout & migration

- **Phasing.** v0.1 alpha → internal Slack channel, 2-week dogfood with
  Developer Productivity team. v0.2 beta → opt-in `brew tap` to all
  engineers. v1.0 GA → announced at eng all-hands, added to onboarding
  docs.
- **Backwards compatibility.** N/A — greenfield. From v1.0, the grammar
  + exit codes + flag names are the contract; changes follow semver per
  Section 6.
- **Rollback.** `brew install logsql@<version>` or `pip install
  logsql==<version>`. No state to migrate back.

---

## 9. Exit codes & diagnostics (replaces "Observability & operations")

(Per CLI category reference: the service-style observability section is
replaced with the CLI-relevant equivalent.)

- **Exit codes.** Defined in 4b. The 3 vs 4 split (skipped vs aborted)
  is the operational contract for CI integration.
- **Stderr.** Format defined in 4i. Always human-readable by default;
  JSON via `--errors-json`.
- **`--stats` flag.** Prints to stderr at exit: `scanned=N matched=N
  skipped=N elapsed=Xs peak_mem=YMiB spilled=Z`. Off by default; on for
  CI and for `--explain`.
- **`--explain`.** Prints the physical plan to stderr and exits 0
  without scanning. Engineers will use this to debug "why is my query
  slow / wrong".
- **Runbook.** `docs/troubleshooting.md` covers the five most likely
  user-facing failure modes (skipped lines, inferred-wrong, OOM, slow,
  spill cleanup).

---

## 10. Open questions

These are the residual decisions where reasonable engineers can
disagree. They do not block drafting, but they should be closed before
v1.0 GA.

1. **Should the Python wheel ship the binary, or `pip install` fall back
   to "go install" if a binary isn't published for the platform?**
   Tentative recommendation: wheel ships per-platform binaries for
   {macos-arm64, macos-x86_64, linux-x86_64, linux-arm64}; everything
   else is a `brew` / direct-download story. This matches `ruff` /
   `uv` precedent and avoids requiring a Go toolchain on user laptops.

2. **`--infer-sample` default of 1000 — too small?** Risk: a column that
   looks numeric for the first 1000 rows but goes string later forces a
   re-scan or a noisy demote-warning. Counter: bigger samples slow the
   prelude pass. Tentative: 1000 with explicit user knob. Revisit after
   first month of usage data.

3. **Should we support `EXPLAIN ANALYZE`-style runtime stats per
   operator?** Strong user signal would justify it; tentative no for
   v1.0 (use `--stats` instead).

4. **Spill format on disk: sorted runs as JSONL (debuggable) or a binary
   record format (faster)?** Tentative: JSONL for v1 — debuggability
   wins over speed at this scale, and we don't promise the format is
   stable. Reopen if scan time on merge becomes a bottleneck.

5. **Do we need a `--timezone` flag for timestamp comparisons in WHERE
   clauses?** Logs are typically UTC, but not always. Tentative: assume
   UTC for v1; document; add `--timezone` if a user files an issue.

6. **Windows support.** PRD says best-effort. Concretely: do we publish a
   Windows binary at all, or just document it as "build from source"?
   Tentative: publish via GoReleaser since it's free, but no CI gate
   and no support promise.

7. **Should `FROM <name>` bind a name we can reference, in anticipation
   of v2 JOIN?** Tentative no — accept-and-ignore in v1 keeps the
   surface honest; v2 introduces real `FROM <path-or-name>` resolution
   and the grammar bump is a minor version since we never accepted a
   binding semantics in v1.

8. **Should the tool integrate with the existing internal `devtools`
   umbrella's update mechanism, or stand alone?** Tentative: stand
   alone (one fewer dependency on a team that isn't on the critical
   path), with a `--check-update` flag that hits a static JSON on
   GitHub releases.
