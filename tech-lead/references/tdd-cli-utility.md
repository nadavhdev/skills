# TDD reference — CLI utility

## When this fits

A command-line tool, one-shot invocation, no long-running process. Reads
inputs (args / stdin / files), produces outputs (stdout / files / exit code),
exits. Examples: `csvkit`-style tools, `kubectl`-style admin tools,
codegen, one-off migrations, dev tooling.

**Not this** if: the tool runs forever, listens on a port, or maintains
in-memory state across invocations — see API serving, event-driven, or
async worker.

## What's different about CLIs

- **Composability is a first-class feature.** The tool will be piped, scripted,
  and called by other tools. STDIN, STDOUT, exit codes, and machine-readable
  output matter more than fancy UX.
- **No retries above you.** When the CLI fails, the human (or CI job) sees
  it directly. Exit codes and error messages *are* the error contract.
- **Bounded memory is a choice you make explicitly.** Streaming vs whole-input
  load is the single biggest design decision. State which you do and why.
- **No observability infrastructure.** No metrics endpoint, no central logs
  (usually). Diagnostics go to stderr in a form humans and CI can read.

## Required sections (in addition to the common skeleton)

Add or rename these in section **4. Detailed design**:

### 4a. Command surface

- Command name, subcommands, args, flags. Show the `--help` output you
  intend to ship (sketched, not actual).
- Input sources: stdin, named files, directories, URLs. Which combinations
  are allowed? What's the default?
- Output destinations: stdout, named file, in-place edit. Default and how
  it's selected.
- Conflicts with existing flags / tools in the same suite (especially
  important if there's a shared CLI framework — e.g. csvkit's common args).

### 4b. Exit codes

Define every non-zero exit code the tool can return, what it means, and
whether scripts can rely on the distinction:

- `0` — success.
- `1` — generic failure (usage error, unexpected exception).
- `2` — argument / usage error (matches `argparse` convention).
- Any tool-specific code (e.g. "differences found" for a diff tool, "no
  matches" for a search tool) — call it out as a deliberate design choice
  and document it.

### 4c. STDIN / pipe behavior

- Does the tool read STDIN when not given a file argument? (It should, for
  pipe composition.)
- Does it handle `tool < file` and `cmd | tool` identically?
- What happens on an interactive TTY with no input — does it hang waiting,
  show help, or error out? (Should error with a useful message.)

### 4d. Memory & streaming

State explicitly whether the tool:
- Streams input row-by-row (bounded memory).
- Loads the entire input into memory (state the bound in the design and in
  user-facing docs — `epilog` if using argparse).
- Buffers internally for a reason (e.g. sort, deduplication, join) — name
  the bound and what knob controls it.

### 4e. Error reporting

- Format for human-readable errors (stderr, prefixed with tool name).
- Format for machine-readable errors if the tool offers a `--json` mode.
- What context is included (input file, row number, column, value).

## NFRs that matter for CLIs

Pull the relevant subset from `nfrs-checklist.md`. For a CLI the
high-signal ones are:

- **Performance** — wall-clock time vs input size. Memory ceiling.
- **Maintainability** — test strategy (especially fixture-based CLI tests).
- **Extensibility** — how do new flags / subcommands fit in without
  breaking existing scripts?
- **Security** — only if the CLI handles secrets, talks to a network
  service, or runs in CI with credentials.

NFRs that usually **do not** apply for a CLI (and should be omitted
explicitly):

- Availability SLO (it's a CLI; there's no uptime).
- Disaster recovery (no persistent state).
- Multi-region (no servers).

## Common pitfalls to call out in the design

- Inventing a new exit code convention that scripts can't predict.
- Reading the whole file when streaming was possible (silent OOM on large
  inputs).
- Output that's only human-readable, when CI / scripts need parseable form.
- Mixing diagnostics into stdout, breaking pipe consumers.
- Crashing on a malformed row instead of skipping with a clear warning
  (or vice versa, when strictness was the right choice).
- Hidden global state (env vars, dotfiles) that makes the same command
  produce different output in different environments.

## Key decisions to surface in "Alternatives considered"

- Streaming vs whole-input load.
- Strict vs lenient input handling.
- Default output (stdout? in-place? new file?).
- Whether this is a new tool or a flag on an existing one in the suite.
