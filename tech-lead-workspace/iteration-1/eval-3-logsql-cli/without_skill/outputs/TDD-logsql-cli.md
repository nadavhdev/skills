# TDD: `logsql` — SQL queries over JSON-lines log files

**Author:** Tech Lead (Developer Productivity)
**Date:** 2026-06-09
**Status:** Draft for review
**Related PRD:** `prd-logsql-cli.md` (Dan Park, 2026-05-28)
**Target ship:** v0.1 in ~6 engineering weeks; v1.0 (man pages, brew tap, signed binaries) ~10 weeks

---

## 1. Problem statement

On-call engineers debug incidents by grepping/jq-ing through `.jsonl(.gz)` log
files exported from CloudWatch Logs Insights. The workflow is slow,
error-prone, and was directly implicated in INC-4421 (40 minutes to correlate
an ID across 12 files). We need a local CLI, `logsql`, that exposes a SQL-like
interface over one or more JSON-lines files, streams gigabyte-scale inputs
without blowing up memory, and produces both human-readable and
machine-readable output.

We are explicitly *not* building a query engine for remote data, a server, or
live-tail. The PRD is unambiguous about that and we will hold the line.

## 2. Scope

### In scope (v0.1)

- Streaming reader for `.jsonl` and `.jsonl.gz` (and STDIN).
- SQL subset: `SELECT ... [FROM <files>] WHERE ... GROUP BY ... HAVING ...
  ORDER BY ... LIMIT ... OFFSET ...`.
- Aggregates: `COUNT`, `COUNT(DISTINCT …)`, `SUM`, `AVG`, `MIN`, `MAX`,
  `APPROX_COUNT_DISTINCT` (HLL).
- Scalar fns: arithmetic, string (`LOWER`, `UPPER`, `SUBSTR`, `LIKE`, regex
  `~`), JSON path (`a.b.c`, `a[0].b`), `COALESCE`, `IF`, `CAST`, time
  (`TIMESTAMP`, `DATE_TRUNC`).
- Output formats: `json` (default; ndjson), `csv`, `tsv`, `table` (aligned),
  `--format=auto` (table if stdout is a TTY, ndjson otherwise).
- Glob expansion at the shell level *and* internally (so users can pass
  `'logs-*.jsonl.gz'` quoted on shells that don't expand).
- Exit codes (see §8).

### Out of scope (v0.1)

- Joins across files (called out in PRD as v2).
- Remote sources (S3/Parquet) — punt to the other tool.
- Live tailing.
- A daemon/server mode.
- Window functions (`OVER (...)`) — large grammar surface; defer to v2.
- `CREATE TABLE`, `INSERT`, `UPDATE` — `logsql` is read-only.
- Subqueries / CTEs — defer to v2. Workaround: shell-pipe two `logsql` calls.

## 3. Decisions (taking a stance)

These are the three contentious choices the PRD flags. I'm calling them now so
implementation can start without bikeshedding.

### 3.1 Language: **Go**

**Decision:** Build in Go. Single statically-linked binary, distributed via
Homebrew tap and GitHub releases (`logsql_<version>_<os>_<arch>.tar.gz`). No
`pip install` path in v1; we will ship `brew install logsql` and a curl-bash
installer.

**Why:**
- The PRD's hard requirement is "productive in under 60 seconds." A static
  Go binary is the lowest-friction install on macOS + Linux. `pip install` on
  a fresh laptop in 2026 is *not* 60 seconds — it routinely drags in compiler
  toolchains for native extensions, runs into PEP 668 "externally-managed"
  errors on system Python, and forces us to pick between `pipx` and `pip`.
- Performance budget (<5 s for 1 GB compressed) is achievable in Python only
  with C extensions (orjson, pyarrow), which then breaks the single-file install
  story. We'd end up shipping a PyOxidizer/Nuitka bundle to get back to Go's
  starting position. Skip the detour.
- Devtools is already a Go shop; we get reuse of internal CI templates,
  release tooling (goreleaser), and code-signing/notarization runbooks.
- Concurrency model fits the workload (one goroutine per file feeding a
  channel into a single executor) far better than Python's GIL-bound async.

**Cost of this decision:** the Python platform team loses the ability to
`import logsql` as a library. Mitigation: we expose a stable JSON output
schema (`--format=json --explain` shows the plan + result schema) so Python
callers can `subprocess.run(["logsql", ...])` and parse. Library mode is
explicitly v2 if there's demand.

### 3.2 Parser strictness: **Forgiving but typed**

**Decision:** Hand-written recursive-descent parser for a *defined* SQL
subset, with **forgiving lexing** (case-insensitive keywords, optional
trailing semicolons, optional `FROM` clause when input comes from files/STDIN)
but **strict semantics** (typed expressions, fail-fast on unknown functions,
no silent string-to-number coercion in comparisons).

**Why not a strict ANSI subset?** Users will paste SQL from Snowflake, BigQuery,
Postgres, and MySQL. Refusing `LIMIT 10` without `OFFSET 0`, or `count(*)`
vs `COUNT(*)`, or `field = 'x'` vs `field = "x"` (we accept both for strings,
because JSON logs themselves are double-quoted) is a UX tax for zero benefit
in a debugging tool.

**Why not "looks like SQL"?** That's how you end up with
`logsql 'select * where status = 200 and message has "foo"'` shipping in week
3 and engineers having to memorize which dialect each tool implements.
Predictability matters more than cleverness.

**Concretely the parser:**
- Accepts: case-insensitive keywords; `'...'` and `"..."` for string literals
  (because shell escaping); `--` and `/* */` comments; trailing semicolons;
  omitted `FROM` (defaults to files passed positionally or STDIN); column
  refs with dotted paths (`payload.user.id`); backticked identifiers for keys
  with special chars (`` `error-code` ``).
- Rejects: anything not in the documented grammar (no silent acceptance of
  unknown clauses). Errors point to the column position in the query string.
- Built using a hand-written Pratt parser (we'll skip ANTLR/goyacc — both add
  build complexity for a grammar this small; ~600 LOC).

### 3.3 Schema: **Schema-on-read by default, optional config**

**Decision:** Default is **schema-on-read with lazy type inference per
column**. We add an *optional* `.logsql.toml` (per-project) that lets users
pin types, declare aliases, and set defaults — but it is never required and
never auto-generated.

**Why:**
- JSON logs are heterogeneous. The same field `latency_ms` may be `int` in
  one service and `float` in another. Forcing users to declare schemas
  up-front kills the "ad-hoc debugging" use case.
- But: experienced users hit pain points (e.g. `timestamp` field that's
  sometimes RFC3339 string, sometimes epoch millis). They deserve an escape
  hatch.

**Inference rules (deterministic, documented):**
- Per-column type starts as `unknown`. First non-null value seen sets it:
  bool / int64 / float64 / string / timestamp (RFC3339 detection) / object /
  array.
- Mixed numeric (int seen, then float seen) → promote to float64.
- Mixed primitive (int seen, then string seen) → fall back to string;
  numeric ops on that column then error unless `CAST(... AS INT)` is used.
- Nested objects and arrays are kept as JSON and accessed via path
  expressions; we do not flatten.

**Optional `.logsql.toml` schema (illustrative):**

```toml
[schema]
"timestamp"   = { type = "timestamp", format = "epoch_millis" }
"latency_ms"  = { type = "float" }
"error-code"  = { type = "string", alias = "error_code" }

[defaults]
format = "table"
```

Resolution order: CLI flags > `./.logsql.toml` > `$XDG_CONFIG_HOME/logsql/config.toml` > built-in defaults.

## 4. High-level architecture

```
+-----------+    +---------+    +---------+    +-----------+    +----------+
|  Sources  | -> | Decoder | -> | Project | -> | Aggregator| -> | Encoder  |
| (files,   |    | (JSON   |    | + Where |    | (optional)|    | (json/   |
|  stdin)   |    |  per    |    |         |    |           |    |  csv/    |
|           |    |  line)  |    |         |    |           |    |  table)  |
+-----------+    +---------+    +---------+    +-----------+    +----------+
      |               |              |               |                |
   bounded         row chan      row chan         row chan         stdout
   readers         (8K buf)      (4K buf)         (1K buf)
   per file
```

Pipeline stages run as goroutines connected by buffered channels. The
**executor is a streaming pull-based iterator tree** built from the parsed
AST. Each operator (Scan, Filter, Project, Aggregate, Sort, Limit) implements
`Next() (Row, error)`. Aggregate and Sort are the only operators that buffer.

### 4.1 Critical path

1. **Plan**: parse → bind (resolve column refs, validate functions) → plan
   (operator tree) → optimize (predicate pushdown into Scan, projection
   pushdown to skip JSON paths we don't need).
2. **Scan**: one goroutine per input file. Decompress (gzip via
   `compress/gzip`), `bufio.Scanner` with a 1 MiB line buffer (configurable
   via `--max-line-size`).
3. **Decode**: streaming JSON decode with `jsoniter` or
   `github.com/goccy/go-json` (decision in §11; both are ~2-3× stdlib).
   *Only* decode the JSON paths the query needs (we walk the AST first to
   collect referenced paths). This is the single biggest perf win.
4. **Filter + Project**: pure CPU, one goroutine, no allocation in the hot
   path (reuse row buffers).
5. **Aggregate** (if GROUP BY or aggregate funcs): hashmap keyed by group
   tuple. Spills to disk when in-memory size > `--mem-limit` (default 1 GiB).
   Spill format = newline-delimited gob; merge via external k-way merge.
6. **Sort** (if ORDER BY): if `LIMIT n` is small, use a heap of size n; else
   external merge sort with the same spill strategy as aggregate.
7. **Encode**: write row-at-a-time to `bufio.Writer` over `os.Stdout`.

### 4.2 Memory model

- Streaming scan → filter → project is O(1) memory regardless of input size.
- `GROUP BY` and `ORDER BY` are bounded by `--mem-limit` (default 1 GiB).
  Beyond that we spill. We **do not silently OOM** — if disk spill is also
  exhausted (`$TMPDIR` < 2 GiB free), we abort with a clear error and a
  pointer to `--mem-limit` and `$TMPDIR`.
- Per-row allocations: target zero. Reusable `Row` struct with column-typed
  fields; JSON values for nested keys stored as `[]byte` slices into the
  scanner's buffer (we copy out only at aggregation or output time).

## 5. Detailed design

### 5.1 Module layout (Go)

```
cmd/logsql/main.go            # CLI entry: cobra
internal/cli/                 # flag parsing, format selection, exit codes
internal/parser/              # lexer, parser, AST
internal/plan/                # logical + physical plan, optimizer rules
internal/exec/                # operators: scan, filter, project, agg, sort, limit
internal/source/              # file glob, stdin, gzip detection
internal/types/               # value type system, coercion rules
internal/encode/              # json/csv/tsv/table encoders
internal/config/              # .logsql.toml loader
internal/spill/               # spill-to-disk for agg and sort
pkg/version/                  # version + build info
```

### 5.2 Type system

Internal value types: `Null | Bool | Int64 | Float64 | String | Timestamp |
Bytes (raw JSON)`. Comparisons:

- Same type: native compare.
- Int vs Float: compare as float64.
- Anything vs Null: comparison returns Null (three-valued logic, like SQL).
- String vs Number: **error**, not silent coercion. Suggest `CAST`.
- Timestamp vs String: parse string as RFC3339; if parse fails, error.

This is the strict half of "forgiving but typed."

### 5.3 SQL grammar (EBNF excerpt)

```
query       := select_stmt
select_stmt := SELECT select_list
               [ FROM source_list ]
               [ WHERE expr ]
               [ GROUP BY expr_list ]
               [ HAVING expr ]
               [ ORDER BY order_list ]
               [ LIMIT int [ OFFSET int ] ]
               [ ';' ]
select_list := '*' | (expr [ AS ident ]) (',' expr [ AS ident ])*
source_list := source (',' source)*
source      := string_literal | ident          ; file path or glob
expr        := or_expr
or_expr     := and_expr (OR and_expr)*
and_expr    := not_expr (AND not_expr)*
not_expr    := [NOT] cmp_expr
cmp_expr    := add_expr ( ('=' | '!=' | '<>' | '<' | '<=' | '>' | '>=' |
                          LIKE | NOT LIKE | '~' | IS [NOT] NULL | IN '(' ... ')')
                          add_expr )?
...
path_expr   := ident ('.' ident | '[' int ']')*
```

Full grammar in `docs/grammar.md` once we hit code review. ~80 productions
total, deliberately small.

### 5.4 Source resolution

- Positional args after the query are treated as file paths or globs.
- If no positional args: read STDIN.
- If STDIN is a TTY and no files: error ("no input — pass files or pipe
  data"), exit 2.
- `.gz` detection: filename suffix *and* magic bytes (`1f 8b`) — we trust
  bytes over suffix.
- Globs expanded by Go's `filepath.Glob` after the shell (for shells that
  don't expand, like when the pattern is quoted).
- Files read in lexicographic order so output is deterministic when no
  `ORDER BY` is present and the user passes `logs-*.jsonl.gz`.

### 5.5 JSON decoding strategy

The biggest perf trap is decoding fields the query doesn't use. Implementation:

1. After parsing, walk the AST and collect the set of referenced JSON paths
   (e.g. `{user.id, request.path, status}`).
2. At scan time, use a streaming JSON tokenizer (`jsoniter` "any" or a custom
   walker) that **only materializes those paths**. Everything else is skipped
   in the tokenizer.
3. For `SELECT *` we still need the whole object, but we keep it as raw
   `[]byte` and only fully decode at output time (output format dependent).

Expected speedup vs. naive `json.Unmarshal` into `map[string]any`: 3–5× on
typical service logs (measured during prototyping; will validate before
freezing the decoder choice).

### 5.6 Aggregation

- Group key = canonical byte-string of the group columns (length-prefixed,
  type-tagged) so equal values produce equal keys regardless of source
  representation.
- Hashmap: `map[string]*aggState` initially. If profiling shows allocation
  pressure we'll switch to a custom open-addressing table; ship the
  `map` version first.
- Spill trigger: when `runtime.MemStats.HeapAlloc` exceeds `--mem-limit` *or*
  when the hash table has > N entries (N=2M default). Spill = sort partitions
  by hash and write gob files; final merge is a k-way merge over those files.

### 5.7 Sort

- If `LIMIT n` is present and small (n ≤ 100k): top-N heap, O(n) memory.
- Else: external merge sort. Same spill machinery as aggregation.

### 5.8 Output encoders

| Format | When chosen | Notes |
|---|---|---|
| `json` | default if non-TTY, or `--format=json` | ndjson, one row per line, UTF-8 |
| `csv`  | `--format=csv` | RFC 4180; nested values JSON-encoded |
| `tsv`  | `--format=tsv` | Tab delimiter, no quoting; nested = JSON |
| `table`| default if TTY, or `--format=table` | aligned columns; buffers up to `--table-buffer` rows (default 200) then flushes, so streaming still works |

`--no-color` disables ANSI; auto-disabled if stdout is not a TTY.

### 5.9 Config file (`.logsql.toml`)

- Optional. Loaded if present in CWD, or in `$XDG_CONFIG_HOME/logsql/`.
- Project file wins over user file (per-key merge, not whole-file).
- CLI flags override config.
- Unknown keys = warning to stderr, not error (forward compat).

### 5.10 Error reporting

- Parse errors: highlight position in the query string with a caret line, like
  `rustc`/`elm`. Example:
  ```
  error: unknown function `LEN`
    SELECT LEN(message) FROM app.jsonl
           ^^^
  hint: did you mean `LENGTH`?
  ```
- Runtime errors (e.g. type mismatch on row 50312): include file path, line
  number, and the offending key. Bad rows are skippable via
  `--on-bad-row=skip|warn|error` (default `warn` to stderr, continue).
- Decode errors on STDIN are treated the same — bad line, byte offset.

## 6. Data model / file formats

`logsql` is stateless and stores nothing of its own except optionally
`.logsql.toml`. No databases, no caches in v0.1.

**Spill files** (transient, under `$TMPDIR/logsql-<pid>-<uuid>/`):
- Gob-encoded streams of `(group_key_bytes, agg_state)` for aggregation, or
  `(sort_key_bytes, row_bytes)` for sort.
- Deleted on process exit (defer + signal handlers for SIGINT/SIGTERM).
- Format is **internal** and not part of any contract; we are free to change it.

## 7. CLI contract (the public API)

```
logsql [flags] <query> [file...]

Flags:
  -f, --format <json|csv|tsv|table|auto>   output format (default: auto)
  -o, --output <path>                       write to file instead of stdout
      --no-header                           CSV/TSV: omit header row
      --color <auto|always|never>           default: auto
      --mem-limit <size>                    e.g. 1GiB, 512MiB (default: 1GiB)
      --max-line-size <size>                default: 1MiB
      --on-bad-row <skip|warn|error>        default: warn
      --explain                             print plan and exit
      --explain-analyze                     run, print plan + per-op stats
  -c, --config <path>                       config file
      --no-config                           ignore config files
  -v, --verbose                             stderr progress (rows/sec)
  -q, --quiet                               suppress non-fatal warnings
      --version
  -h, --help
```

### 7.1 Stable contracts

- **JSON output schema**: ndjson, one object per row. Column names = SELECT
  aliases (or expression text if unaliased). Nested values are emitted as
  JSON, not strings. **This is the contract for scripted callers.**
- **`--explain` output**: indented plan tree, machine-parseable with
  `--explain --format=json`.
- **Exit codes** (see §8).
- **Flag names**: stable from v0.1. We can *add* flags; we don't remove or
  rename in 0.x without a deprecation cycle.

## 8. Exit codes

| Code | Meaning |
|---|---|
| 0 | Success, at least zero rows produced |
| 1 | Runtime error (I/O, unparseable JSON beyond `--on-bad-row` tolerance) |
| 2 | Usage error (bad flags, bad query, no input on TTY) |
| 3 | Resource limit hit (mem-limit + spill exhausted) |
| 130 | SIGINT |

Deliberately no "exit 1 if zero rows" — that's a `grep` quirk that doesn't fit
SQL semantics (`SELECT 0` is a valid result).

## 9. Non-functional requirements

### 9.1 Performance

PRD target: < 5 s time-to-first-result for 1 GB compressed (~5-10 GB
decompressed) on a 2024 MacBook Pro.

**Budget breakdown for `SELECT COUNT(*) WHERE service='x'` on 1 GB gzip:**

| Stage | Budget | How we hit it |
|---|---|---|
| Gzip decompress | 2.0 s | `compress/gzip` saturates ~500 MB/s decompressed on M-series |
| Line scan | 0.4 s | `bufio.Scanner`, 1 MiB buffer |
| JSON tokenize + extract `service` | 1.5 s | path-targeted decode (§5.5) |
| Filter + count | 0.3 s | trivial |
| Output | 0.01 s | one row |
| **Total** | **~4.2 s** | with ~15% headroom |

We will measure with `pprof` and `benchstat` before committing to this. If
gzip dominates beyond budget, we'll evaluate `klauspost/pgzip` (parallel
gzip), which is API-compatible — that's a 1-line swap.

### 9.2 Capacity

- Designed for **interactive** workloads: 1 user, 1 invocation, typically
  ≤ 10 GB compressed input.
- Concurrent invocations on the same machine: no shared state, no locks; each
  process is independent and bounded by its own `--mem-limit`.
- Spill: bounded by `$TMPDIR` free space. Document this.

### 9.3 Reliability

- No daemon, no state — "reliability" reduces to "does it crash on weird
  input?" Strategy:
  - Fuzz the parser (`go test -fuzz`) for ≥ 1 hour in CI per release.
  - Fuzz the JSON line reader with truncated, multi-MB, and non-UTF8 input.
  - Property tests for aggregation correctness against a naive reference
    implementation on small inputs.
- Graceful SIGINT: flush partial table-format output, delete spill files,
  exit 130.

### 9.4 Observability

- `--verbose`: emit rows/sec, bytes/sec, current file, every 1 s to stderr.
- `--explain-analyze`: per-operator wall time, rows in/out, bytes spilled.
- `LOGSQL_PPROF=cpu logsql ...` enables a `pprof.StartCPUProfile` dump to
  `./logsql.cpu.pprof` (dev/diagnostic only, not advertised).
- No telemetry, no phone-home. We are a local CLI; we will not collect usage
  data in v0.1. If we want metrics later, it goes behind an opt-in flag with
  a privacy review.

### 9.5 Security

- **Untrusted input**: log files may contain attacker-controlled data
  (request bodies, user-agents). We must not:
  - Execute anything from JSON (no `eval`, no template expansion).
  - Allocate unboundedly from a single line: enforce `--max-line-size`
    (default 1 MiB), abort the line and warn on overflow.
  - Allow path traversal via the query (queries reference *columns*, not
    files; file args are resolved by the OS).
- **Binary distribution**: signed and notarized for macOS; checksums + cosign
  signatures for Linux releases. SLSA level 2 via goreleaser.
- **No network**: assert in CI that the binary has no DNS/syscalls beyond
  file I/O (a smoke test that runs under `unshare -n` on Linux).
- **CVE response**: dependencies are listed in `go.mod`; we have `dependabot`
  enabled. SLA: patch high-CVE deps within 7 days.

### 9.6 Portability

- macOS arm64 + amd64, Linux amd64 + arm64: tier-1, full CI matrix.
- Windows amd64: tier-2, best-effort (PRD allows this). Known issues: TTY
  detection and ANSI colors via `golang.org/x/term` + `mattn/go-isatty`;
  path globbing semantics differ slightly.
- Go version: 1.22+ (for `slices`, `maps`, `log/slog`).

### 9.7 Accessibility / DX

- All errors go to stderr; never stdout.
- Output is line-buffered in TTY mode, fully buffered in pipe mode (with a
  reasonable flush interval) — same as `grep --line-buffered`.
- `--help` fits in 40 lines for the top-level command; subcommands not used
  in v0.1.

## 10. Alternatives considered

### 10.1 Language

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Go** | Single static binary; fast; team expertise; goreleaser | Slightly less ergonomic for users who want to embed it | **Chosen** |
| Python | Easier to extend; familiar to platform team; fast iteration | `pip install` UX in 2026 is bad; perf requires C deps; bundling adds complexity | Rejected — install friction kills the "60-second" promise |
| Rust | Fastest; smallest binary; great parser libs (chumsky, nom) | Team has limited Rust expertise; longer build/CI; harder to onboard contributors | Rejected — not enough delta over Go to justify the team cost |
| DuckDB CLI as a wrapper | Free SQL engine, fast, handles JSON | Heavy dep (~50 MB), wrong defaults for ndjson logs (treats files as tables, not streams), can't do path-targeted decode the way we want, license/distribution complexity | Rejected — but we'll steal ideas from its planner |
| Use `q` / `jq` / `dsq` | Already exist | None fit: `q` is CSV-first, `jq` isn't SQL, `dsq` requires schema and isn't streaming | Rejected as a no-build option |

### 10.2 Parser

- **ANTLR-generated parser**: too much build-time tooling; ~10× the binary
  surface; debugging generated code is awful. Rejected.
- **goyacc**: more idiomatic for Go but generated code is ugly and we lose
  good error messages. Rejected.
- **PEG (e.g. pigeon)**: nice for prototyping but locking us into another
  codegen step. Rejected.
- **Hand-written Pratt parser**: ~600 LOC, full control over error
  messages, no codegen. **Chosen.**

### 10.3 Schema

- **Mandatory config**: rejected — kills ad-hoc use.
- **Auto-generated schema from first 1000 rows, cached**: tempting, but the
  cache invalidation rules are subtle and the cache becomes a debug
  liability ("why are my types wrong? oh, the cache"). Rejected.
- **Pure schema-on-read, no escape hatch**: rejected — leaves users stuck
  on the epoch-vs-RFC3339 timestamp problem.
- **Schema-on-read with optional `.logsql.toml`**: **Chosen** (§3.3).

### 10.4 Execution model

- **Materialize everything, then query**: would blow the 5 GB memory target
  on big files. Rejected.
- **Vectorized (columnar batches)**: faster for analytical queries; more
  complex to implement; benefits less obvious for ndjson row-at-a-time
  inputs. Defer to v2 if profiling shows row-at-a-time is the bottleneck.
- **Row-at-a-time pull iterators** (Volcano-style): **Chosen.** Simple,
  composes, well-understood, fits streaming.

## 11. Open questions

Things the team needs to decide *during* implementation, not before:

1. **JSON decoder**: `jsoniter` vs `goccy/go-json` vs hand-rolled tokenizer.
   Bench all three on real log fixtures in week 1; pick by p50 ms/MB.
2. **`LIKE` semantics**: SQL standard (`%`, `_`) or shell-style (`*`, `?`)?
   Lean toward SQL standard with `--like-shell` opt-in; confirm with two
   on-call engineers.
3. **Approximate count distinct**: HLL precision tradeoff — `precision=14`
   (~1.6% error, 16 KiB per group) is the usual default; confirm that's
   acceptable.
4. **Timestamp inference aggressiveness**: should `"2026-06-09"` (no time
   component) be inferred as `Date` or `String`? Lean `String` to avoid
   surprises; revisit after dogfooding.
5. **Brew tap location**: own tap (`devtools/homebrew-tap`) vs homebrew-core.
   Probably own tap first; consider core after v1.0 stability.
6. **Telemetry policy**: if and when we add opt-in usage stats, what do we
   collect and where does it go? Punt to a separate doc; not in v0.1.
7. **Path expression edge cases**: how do we reference a key literally named
   `*` or one containing a dot? Current plan: backticks (`` `weird.key` ``);
   confirm with the parser implementer.
8. **Config discovery walk**: do we walk *up* from CWD looking for
   `.logsql.toml` (git-style) or only check CWD? Lean "only CWD + user
   config" for predictability.

## 12. Risks & failure modes

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| 5 s perf target missed on 1 GB gzip | Med | High (PRD requirement) | Prototype week 1 with the path-targeted decoder; if we miss, evaluate `pgzip` and SIMD JSON (`simdjson-go`) before scope-cutting features |
| Hand-written parser becomes a maintenance burden | Med | Med | Keep grammar small (documented EBNF, ≤ 80 productions); high test coverage on parser; revisit codegen at v2 if grammar grows |
| Users expect joins / subqueries | High | Low (PRD explicitly v2) | Error messages on `JOIN`/subquery point to the v2 issue and suggest shell-pipe workarounds |
| Spill files leak on hard crash | Low | Low | `os.TempDir()` cleanup on every startup of files matching `logsql-*-*` older than 24h |
| Win32 path/TTY bugs delay release | Med | Low | Windows is best-effort per PRD; CI runs Windows tests but failures are non-blocking for v0.1 |
| Heterogeneous types in same column produce confusing errors | High | Med | Crisp error message: "column `latency_ms` is mixed (int and string at line 50312); use `CAST(latency_ms AS FLOAT)` to coerce" |
| Distribution / signing keys lost | Low | High | Keys in 1Password vault, mirrored to break-glass; signing runbook in `RUNBOOK.md` |
| Devs paste a query with a typo and `WHERE col` matches nothing because of strict types | Med | Med | `--verbose` summary: "0 rows matched out of N; skipped M rows due to type errors on column X" |

## 13. Rollout & migration

There's nothing to migrate from — this is a new tool. Rollout plan:

### Phase 0 — week 1
- Scaffolding, parser+lexer, AST, JSON path decoder.
- Run perf prototype on real fixtures (10×1 GB compressed logs from staging).
- **Gate**: hit ≤ 6 s on the prototype (10% slop on the 5 s target) or escalate
  before continuing.

### Phase 1 — weeks 2-4 — Alpha (v0.1.0)
- Full operator set: Scan, Filter, Project, Aggregate, Sort, Limit.
- Output formats: json, csv, table.
- `--explain`.
- Internal release: `go install` from the repo for the devtools team.
- Dogfood with 5 on-call engineers; collect feedback via a single channel.

### Phase 2 — weeks 5-6 — Beta (v0.5.0)
- TSV output, color, `--explain-analyze`.
- `.logsql.toml` config.
- Spill-to-disk for agg and sort.
- Homebrew tap (`devtools/tap`), Linux tar.gz on GitHub releases.
- Open beta: announce in `#dev-productivity`, invite all engineers.

### Phase 3 — weeks 7-10 — GA (v1.0.0)
- Fuzzing in CI (parser + JSON reader, ≥ 1 hour per release).
- Signed/notarized macOS binary; SLSA-2 provenance for Linux.
- Man page (`man logsql`).
- Docs site (just a single-page README + grammar reference).
- INC-4421 retro action item closed; pair with on-call to do a re-run of
  the original incident scenario using `logsql` and capture the time
  difference for the retro record.

### Rollback
- It's a CLI; rollback = users keep the previous version installed. Homebrew
  pins via `brew install logsql@0.5`. We will keep release artifacts forever
  on GitHub releases.

### Success metrics
- Adoption: ≥ 30 weekly active users by week 10 (measured via `--verbose`
  invocations submitted in incident retros, not telemetry — opt-in only).
- Performance: ≥ 90% of invocations in the dogfood logs finish in < 10 s.
- INC retros mentioning "log search time" drop by 50% in the next two
  quarters.
- DX survey: "log debugging on local machines" drops out of top-5 pain
  points in the next survey.

## 14. Testing strategy

- **Unit**: every package has tests; parser has ≥ 200 golden-query cases
  covering happy paths and every error message.
- **Integration**: `testscript`-style `.txtar` tests under `testdata/scripts/`
  that drive the binary end-to-end on fixture log files. CI runs the same
  suite on macOS / Linux / Windows.
- **Property**: random query generator + reference impl (literally a Python
  script reading the same ndjson) for cross-check on small inputs.
- **Fuzz**: parser fuzzer and JSON-line fuzzer in CI; corpus checked into
  the repo.
- **Perf**: `benchstat` on a fixed 1 GB corpus, results posted to a GitHub
  comment on every PR. Regressions > 10% on the benchmark suite block merge.

## 15. Appendix

### 15.1 Example queries (acceptance)

```
# A1: find error correlation
logsql 'SELECT timestamp, correlation_id, message
        WHERE error_code = "ECONN" AND service = "billing"
        ORDER BY timestamp' service-*.jsonl.gz

# A2: noisy users
logsql 'SELECT user_id, COUNT(*) AS n
        GROUP BY user_id
        ORDER BY n DESC
        LIMIT 10' app.jsonl

# A3: pipe-friendly
logsql --format=json 'SELECT path, COUNT(*) AS n
                      GROUP BY path
                      ORDER BY n DESC
                      LIMIT 20' \
  < huge.jsonl | jq '.path'

# A4: stdin from gunzip
gunzip -c huge.jsonl.gz | logsql 'SELECT path, COUNT(*)
                                  GROUP BY path
                                  ORDER BY 2 DESC
                                  LIMIT 20'
```

### 15.2 Things explicitly *not* in v0.1

For the reviewer's quick scan, also see §2:
- JOINs, subqueries, CTEs, window functions.
- Remote sources (S3, HTTP, Parquet).
- Live tailing.
- Server/daemon mode.
- Library / `import logsql` use from Python.
- Custom output templates beyond the 4 supplied formats.

### 15.3 Dependency budget

Direct deps cap: **≤ 10**. Initial list:
- `github.com/spf13/cobra` (CLI)
- `github.com/spf13/pflag` (transitive)
- `github.com/BurntSushi/toml` (config)
- `github.com/mattn/go-isatty` (TTY detection)
- `golang.org/x/term` (terminal width)
- one of {`github.com/json-iterator/go`, `github.com/goccy/go-json`} (TBD §11.1)
- `github.com/klauspost/compress` (optional, faster gzip)
- `github.com/google/uuid` (spill dir naming)

No ORMs, no test frameworks beyond `testing` + `testscript`.
