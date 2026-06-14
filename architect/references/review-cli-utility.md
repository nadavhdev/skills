# Architect review aspects — CLI utility

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-cli-utility.md` first — that is
the standard this TDD was (or should have been) written against. Use its
"Required sections", "NFRs that matter / don't apply", and "Common pitfalls" as
the baseline checklist the design must satisfy. This file adds the architect
lens on top: the decisions to scrutinize hardest and the calibration to enforce.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **Exit-code convention.** Scripts and CI will encode these. A tool-specific
  code (e.g. "differences found", "no matches") is a contract; adding or changing
  one later breaks every consumer. Is the full table defined and deliberate?
- **Output format / contract.** Once tools and pipes parse the output, its shape
  is locked. Is there a machine-readable mode where consumers need one?
- **Flag names and defaults.** Renaming a flag is a breaking change. Do they
  collide with an existing shared-CLI framework's flags (e.g. csvkit common args)?
- **New tool vs. a flag on an existing one.** Splitting or merging later is
  disruptive. Did they justify standing up a new tool?

## Type-specific hot spots (review questions)
- **Memory: streaming vs whole-load.** The single biggest CLI decision. Is it
  stated and right for the expected input size? A whole-file load with no stated
  bound is a silent-OOM risk → `risk`/`scalability` finding.
- **STDIN / pipe behavior.** Does it read STDIN when no file is given? Behave
  identically for `tool < file` and `cmd | tool`? Error usefully on an
  interactive TTY with no input rather than hanging?
- **Error channel discipline.** Diagnostics on stderr, data on stdout? Mixing
  them breaks pipe consumers (`observability`/`contract`).
- **Strict vs lenient input.** Crash on a malformed row vs skip-with-warning —
  is the choice deliberate and matched to the use case?

## Over-engineering & simplification tells
- A plugin/extension framework for what is one tool with one job.
- A config-file system where a handful of flags would do.
- A daemon / server mode nobody asked for ("for speed") — that's a different
  workload; point back at the CLI shape.
- Reinventing arg parsing / output formatting the suite's shared framework
  already provides (`codebase-fit`).

## Calibration — NFRs
- **Flag as over-reach if present:** availability SLO, disaster recovery,
  multi-region — a CLI has no uptime and no persistent state. Their presence is
  a miscalibration worth a `minor` `over-engineering` finding.
- **Demand if absent:** an explicit memory bound, the exit-code table,
  STDIN/pipe behavior, and a **test strategy** (fixture-based CLI tests that
  cover each exit code and the streaming path). These are the CLI must-haves.
