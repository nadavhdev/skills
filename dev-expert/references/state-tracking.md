# State tracking — `state.json` schema and `task-XX.done.md` template

This file defines the three artifacts dev-expert maintains for every task:

1. **`task-XX.state.json`** — machine-readable. Tracks where the
   pipeline is, so any interrupted run can resume. Lives next to the
   task file.
2. **`task-XX.review.md`** — human-readable review ledger. The
   round-by-round record of the review loop, with the code evidence and
   the fix for each finding. Lives in the tasks dir root. Written by the
   `review` stage, one section appended per round.
3. **`task-XX.done.md`** — human-readable handoff. Tells future tasks
   what was actually built, what decisions were made, and what gotchas
   to know. Lives in the tasks dir root (even after the task file is
   moved to `done/`).

All three are part of the deliverable. Skipping any one breaks the
workflow's resumability, its auditability, or the next task's ability to
build on this one.

---

## `task-XX.state.json` schema

The file is named after the task it tracks. For
`./tasks-csvdiff/01-walking-skeleton-keyed-csvdiff.md`, the state file is
`./tasks-csvdiff/01-walking-skeleton-keyed-csvdiff.state.json`.

```json
{
  "task_id": "01-walking-skeleton-keyed-csvdiff",
  "task_path": "./tasks-csvdiff/01-walking-skeleton-keyed-csvdiff.md",
  "current_stage": "review",
  "stages_completed": ["load", "implement", "tests", "verify"],
  "started_at": "2026-06-10T16:42:11Z",
  "updated_at": "2026-06-10T17:18:03Z",
  "completed_at": null,
  "attempts": {
    "load": 1,
    "implement": 1,
    "tests": 1,
    "verify": 2,
    "review": 1
  },
  "last_error": null,
  "files_changed": [
    "csvkit/utilities/csvdiff.py",
    "tests/test_utilities/test_csvdiff.py",
    "examples/diff_a.csv",
    "examples/diff_b.csv",
    "pyproject.toml"
  ],
  "verify_commands": [
    {"cmd": "pytest --cov csvkit", "passed": true, "ts": "2026-06-10T17:10:01Z"},
    {"cmd": "flake8 .", "passed": true, "ts": "2026-06-10T17:10:31Z"},
    {"cmd": "isort . --check-only", "passed": true, "ts": "2026-06-10T17:10:33Z"},
    {"cmd": "check-manifest", "passed": true, "ts": "2026-06-10T17:10:55Z"}
  ],
  "review_rounds": [
    {
      "round": 1,
      "diff_sha": "abc1234",
      "verdict": "REQUEST_CHANGES",
      "findings": [
        {
          "severity": "major",
          "area": "exit codes",
          "summary": "parse errors must wrap to exit 2 at the from_csv site, not exit 1 default",
          "addressed": true,
          "addressed_with_modification": false
        },
        {
          "severity": "minor",
          "area": "test coverage",
          "summary": "missing test for the four-invocation-style stdin matrix",
          "addressed": true,
          "addressed_with_modification": false
        }
      ],
      "rebuttals": [
        {
          "finding_summary": "<finding the implementer rebutted>",
          "reviewers_claim": "<the reviewer's detail, condensed>",
          "evidence_checked": [
            "csvkit/utilities/csvdiff.py:120-140",
            "tasks-csvdiff/01-walking-skeleton-keyed-csvdiff.md (acceptance criterion 4)",
            ".claude/CLAUDE.md (Exit codes section)"
          ],
          "why_claim_does_not_hold": "<2-4 sentences with concrete reason>",
          "action_taken": "no change"
        }
      ],
      "prior_findings_review": [
        {"ref": "1.1", "status": "closed", "note": "fix added the missing error-path test"},
        {"ref": "1.2", "status": "reopened", "note": "rename only renamed the call site, not the definition"}
      ],
      "ts": "2026-06-10T17:18:03Z"
    }
  ],
  "surfaces_raised": [
    {
      "stage": "implement",
      "question": "Spec calls for X but Y issue arises. Options: (a) ship as spec, (b) modify with Z.",
      "options": ["a", "b", "c"],
      "user_choice": "a",
      "ts": "2026-06-10T16:55:00Z"
    }
  ],
  "self_check": {
    "stage": "implement",
    "ts": "2026-06-10T17:05:00Z",
    "items": [
      {"item": "clean names", "status": "yes", "note": null},
      {"item": "single responsibility", "status": "yes", "note": null},
      {"item": "SOLID seams", "status": "yes", "note": null},
      {"item": "decoupling", "status": "yes", "note": null},
      {"item": "reuse", "status": "yes", "note": null},
      {"item": "readability", "status": "yes", "note": null},
      {"item": "extendability", "status": "yes", "note": null},
      {"item": "maintainability", "status": "yes", "note": null},
      {"item": "security at boundaries", "status": "yes", "note": null},
      {"item": "error handling", "status": "yes", "note": null},
      {"item": "no PII in logs", "status": "yes", "note": null},
      {"item": "scope discipline", "status": "yes", "note": null}
    ],
    "deferred": []
  },
  "branch": "feat/csvdiff-walking-skeleton",
  "branch_strategy": "new",
  "branch_base": "master",
  "branch_decided_at": "2026-06-10T16:48:11Z",
  "review_gate_preference": "conditional",
  "commit_sha": null,
  "pr_url": null,
  "review_skipped": false,
  "dependencies_verified": [],
  "prior_done_md_read": [],
  "tdd_path": "./TDD-csvdiff.md",
  "tdd_sections_read": ["§0", "§3", "§4a", "§4b", "§4c", "§4d", "§4e", "§4f", "§4g", "§4h", "§6", "§7", "§10"],
  "tdd_correlation": {
    "exit code 0 when no diff": "§4b Exit codes",
    "parse errors exit 2": "§4b Exit codes + §7 Risks",
    "four invocation styles": "§4c STDIN/pipe behavior",
    "single-column -c resolution": "§4g Comparison semantics",
    "--ignore drops columns": "§4g Comparison semantics",
    "perf-smoke 200k rows": "§6 Scalability"
  }
}
```

### Field semantics

| Field                  | Purpose                                                                                                  |
|------------------------|----------------------------------------------------------------------------------------------------------|
| `task_id`              | Stem of the task filename (no extension). Used as a stable identifier.                                   |
| `task_path`            | Resolved absolute or workspace-relative path to the task `.md`. Useful for resume.                       |
| `current_stage`        | One of `load`, `branch`, `implement`, `tests`, `verify`, `review`, `checkpoint`, `commit`, `pr`, `handoff`, `done`. The single source of truth for "where am I." |
| `stages_completed`     | Ordered list of stages already passed. Lets you tell at a glance how far you got.                        |
| `attempts.<stage>`     | How many tries each stage took. High `verify` attempts → lots of fix cycles. High `review.rounds` → diverging review. |
| `last_error`           | `null` on success, or `{stage, message, ts}` if a stage attempt failed and is being retried.             |
| `files_changed`        | Running list of files touched. The `commit` stage uses this to know what to stage.                       |
| `verify_commands`      | Last run of each verify command with pass/fail + timestamp. Useful diagnostics if verify drifts later.   |
| `review_rounds`        | One entry per review iteration. Each entry holds `findings` (with `addressed` + `addressed_with_modification` flags), `rebuttals` (evidence-based pushbacks per `references/review-loop.md`), and — on rounds ≥ 2 — `prior_findings_review` (the reviewer's `closed`/`not_closed`/`reopened` status for each prior finding under the targeted re-review). This is the machine record; the human-readable counterpart with code evidence and fix diffs is the ledger `task-XX.review.md`. Critical for auditability of the review loop. |
| `surfaces_raised`      | List of pre-deviation surfaces — moments where the implementer paused mid-stage to ask the user whether to ship-as-spec, modify, or revise the TDD. Per the prime directive, no silent deviation allowed. |
| `self_check`           | The implementer's pre-exit-`implement`-stage walkthrough of the persona checklist (clean code, SOLID, decoupling, reuse, readability, extendability, maintainability, security at boundaries, error handling, no PII in logs, scope discipline). Every item has `status: yes/no/N-A` + optional note. The reviewer audits this; an item marked "yes" that the reviewer disproves is a `code-quality`/`security`/`scope` finding. |
| `review_skipped`       | `true` only if the user explicitly asked to skip review. Surfaced in the PR description.                 |
| `branch`, `branch_strategy`, `branch_base`, `branch_decided_at` | Filled in by the `branch` stage. `branch_strategy` is `"new"` (new branch created off `branch_base`) or `"continue"` (stayed on the prior branch because a dep's unmerged commits were only reachable there). `branch_base` is set only when `branch_strategy: "new"` — it records which canonical base the new branch was forked from (typically `master` or `main`). |
| `review_gate_preference` | Set once by the `branch` stage from a session-start question: `"always"`, `"conditional"` (default), or `"never"`. Governs whether the `checkpoint` stage pauses before the PR for the user to inspect the review ledger. In non-interactive mode it is left at the default `"conditional"` (no human to ask) — the `checkpoint` stage still never pauses without a human, but records whether the gate would have fired. |
| `commit_sha`, `pr_url` | Filled in by the `commit`, `pr` stages.                                                                                 |
| `dependencies_verified`| Task ids whose done.md was confirmed present during `load`.                                              |
| `prior_done_md_read`   | Paths of all done.md files actually read into context during `load`. Audit trail.                       |
| `tdd_path`             | Absolute or workspace-relative path to the parent TDD found during `load`. `null` is **not** valid — if the TDD cannot be found, the workflow refuses to start. |
| `tdd_sections_read`    | Specific TDD sections (e.g. `§4b`) the implementer pulled into context. Lets review audit that the right sections were read for this task. |
| `tdd_correlation`      | Map from each acceptance criterion (short label) to the TDD section(s) it derives from. Built during the `implement` stage; the review subagent verifies it. A criterion with `null` correlation is a yellow flag and must be surfaced. |

### Write rules

- Update `updated_at` on **every** write.
- Update `state.json` at the **start** and **end** of each stage:
  - Start: bump `attempts.<stage>` by 1.
  - End: append to `stages_completed`, advance `current_stage`.
- On a stage failure mid-flight, write `last_error` and **do not**
  advance `current_stage`. The next run can pick up the same stage.
- Never rewrite past `review_rounds` entries — append only.

---

## `task-XX.done.md` template

The done.md is the **ground truth for future tasks**. It is the answer
to "what did the previous person actually build, and what did they
learn?" Keep it short enough that the next task reads it without
skimming, but specific enough that it answers concrete questions.

Lives in the tasks dir **root**, not in `done/`. That way the next
task's `load` stage finds it without having to scan subdirectories.

### Template

```markdown
# task-XX — done

**Task spec:** [link to the moved file in done/](done/XX-<slug>.md)
**PR:** <url>
**Commit:** <sha>
**Completed:** <ISO date>
**Branch:** <branch name>

## What was built

<2-5 sentences. Plain English. What does the user-facing surface look
like now that didn't before? Name the public API, the CLI flags, the
files added.>

## Files changed

- `path/to/file1.py` — <one line: created / added X behavior>
- `path/to/file2.py` — <one line: modified to do Y>
- `tests/path/test_X.py` — <one line: covers A, B, C>
- ... (every file touched, with a one-line "why")

## Decisions & departures from spec

<List of decisions made that weren't fully specified in the task .md,
and any places where the implementation deviated from the literal text
of the spec — with reasoning. This is the part the next task needs
most. If there were none, write "None — implementation matches spec
exactly.">

- **Decision X:** chose <option> because <reason>. Alternatives
  considered: <list>.
- **Departure Y:** the spec said <X> but the implementation did <X'>
  because <reason>. <Impact on downstream tasks, if any.>

## Test coverage

<What's covered and what's deliberately not. Don't make this exhaustive
— call out the non-obvious ones.>

- ✓ <behavior 1>
- ✓ <behavior 2>
- ✗ <thing we intentionally didn't test, with reason>

## Review findings & resolutions

<One-paragraph summary only. The full round-by-round detail — with the
code evidence and the fix diff for every finding — lives in the review
ledger; link to it and do not duplicate it here.>

**Full ledger:** [task-XX.review.md](task-XX.review.md)

- Round 1: <N findings, severity distribution>. Resolved by <summary>.
- Round 2 (if applicable): ...
- Deferred nits: <list, or "none">.

## Things the next task should know

<The single most valuable section. Forward-looking notes for whoever
picks up the next task in the dependency chain.>

- The function `<name>` in `<file>` is the seam where the next task
  should hook in for <X>.
- Naming convention: I used `<X>` — please mirror it for consistency.
- Don't <Y> because <Z> — I tried it and it broke <W>.
- Known limitation: <X>. Out of scope for this task; tracked as
  `<deferred-thing>`.

## Open questions surfaced

<Any questions that came up that weren't answered by the spec and
that downstream tasks may need to resolve. If none, write "None.">
```

### Why this template is shaped like this

- **"What was built"** gets a future reader oriented in five seconds.
- **"Files changed"** is for grep-style navigation — the next task often
  wants to know "where is the function X lived" without re-reading the
  diff.
- **"Decisions & departures from spec"** is the **load-bearing section**.
  This is what a future tech-lead-review of the next task will check
  against. It is also what protects you from the next task accidentally
  undoing a deliberate choice.
- **"Review findings"** preserves the audit trail without forcing the
  reader through state.json.
- **"Things the next task should know"** is unique to this skill — it's
  the part that makes the handoff useful as ground truth instead of as
  archaeology.

### Anti-patterns to avoid in the done.md

- A play-by-play of every commit. Use git for that.
- "Refactored for clarity" with no specifics — say *what* changed and
  *why*.
- Marketing prose ("seamlessly integrated", "robust solution"). Be
  technical and concrete.
- Repeating the task spec verbatim. The spec is in `done/` if anyone
  needs it.

---

## `task-XX.review.md` — the review ledger

The review ledger is the **human-viewable, round-by-round record of the
review loop**, carrying the code evidence and the fix for each finding.
It exists because neither `state.json` (machine JSON, no code context)
nor `done.md` (a terse end-state summary) lets a human *see* what each
round flagged, whether the implementer pushed back, and exactly what the
fix was.

The ledger is also the **carrier of incremental review context**: on
review rounds ≥ 2, the reviewer reads it to learn what was already found
and changed, so it can scope down to a targeted re-review of the fixes
plus a regression scan, instead of re-deriving every finding from
scratch (see `references/review-loop.md`).

Lives in the tasks dir **root** as `task-XX.review.md`. dev-expert
**appends one section per review round, at two moments**:

1. **Right after the reviewer returns a verdict** — write the round
   header, the verdict, the reviewer summary, and every finding with its
   code evidence (and, on rounds ≥ 2, the prior-finding closure statuses
   the reviewer reported).
2. **Right after you apply / rebut / defer the findings** — fill in the
   Resolution + Fix block under each finding: the fix as a diff, or the
   evidence-based rebuttal, or the deferral reason.

Append-only — **never rewrite a past round's section**, exactly like
`state.json.review_rounds`. The ledger and `review_rounds` are written
from the same data in the same step; the ledger adds the code evidence
and fix diff that JSON can't hold. If they ever disagree, `state.json`
is the machine source of truth and the ledger is the human view — keep
them consistent.

### Template

The template below is shown inside a four-backtick fence so its own
three-backtick code blocks render; the ledger file itself uses normal
three-backtick fences.

````markdown
# Review ledger — task-XX

**Task spec:** [done/XX-<slug>.md](done/XX-<slug>.md)
**Reviewer:** tech-lead implementation-review capability (a fresh subagent each round)

---

## Round 1 — <APPROVE | REQUEST_CHANGES> (<n> findings: <b> blocker, <m> major, <mi> minor, <ni> nit)

**Scope:** full review (`git diff <merge-base>..HEAD`)
**Reviewer summary:** <the subagent's `summary` field, verbatim>

### Finding 1.1 — [<severity>] <area> — <one-line summary>
**Anchor:** <spec_anchor — acceptance criterion / TDD section / done.md decision, or "general">
**What the reviewer said:** <the finding's `detail`, condensed to 1–3 sentences>

**Code it points at:**
```python
# <file>:<line>
<the 3–12 lines the finding is about — the actual code, copied from the file>
```

**Challenge:** <one of:
- "None — finding holds." (you simulated it per review-loop.md Step A and it's valid), or
- an evidence-based rebuttal mirroring state.json.rebuttals: what you
  checked, why the claim does not hold, what you're doing instead>

**Resolution:** <applied | applied-with-modification | rebutted (no change) | deferred>
**Fix:**
```diff
<the actual change you made — the diff hunk; or "n/a — rebutted" / "n/a — deferred to <where>">
```

### Finding 1.2 — ...

---

## Round 2 — <verdict> — targeted re-review (<n> new findings)

**Scope:** targeted re-review — verified prior findings, deep-reviewed the round-1 fixes, regression-scanned the rest.
**Prior findings status (reported by reviewer):**
- 1.1 — <closed | not_closed | reopened> — <reviewer note>
- 1.2 — <closed | ...>

**Reviewer summary:** <verbatim>

### Finding 2.1 — ...
<Only *new* findings (including any regression a fix introduced, or a
prior finding the reviewer reopened) get a full entry here. Priors the
reviewer marked `closed` are recorded in the status list above, not
repeated as findings.>

---

## Outcome

**Final verdict:** APPROVE at round <N>
**Deferred (nits / accepted rebuttals):** <list, or "none">
````

### Field rules

- **Code evidence is mandatory for every finding** that names a file —
  copy the actual lines from the file (not a paraphrase), with a
  `# <file>:<line>` header. A finding with no code evidence is only
  acceptable when its `file` is `null` (a cross-cutting finding).
- **The Fix block is a real diff**, not a description. Paste the hunk you
  changed. For a rebutted finding, write `n/a — rebutted` and rely on
  the Challenge block; for a deferred nit, write `n/a — deferred to
  <handoff "Deferred nits" / follow-up>`.
- **Rounds ≥ 2 do not repeat closed priors.** The reviewer's
  per-prior-finding status list is the record that they were verified;
  re-listing them as findings bloats the ledger and hides the new
  signal.
- **The Outcome block is written once, at the very end** — when the
  `checkpoint` stage clears to `commit`, not at the first APPROVE. A
  pre-PR pause can add more review rounds (changes the user requested
  during the checkpoint), so the final round number isn't known until
  the checkpoint releases.

---

## File layout summary

After the `handoff` stage completes, the tasks dir looks like this:

```
./tasks-csvdiff/
├── README.md                                       # the breakdown index
├── 02-composite-key-and-duplicate-handling.md      # not yet started
├── 03-no-key-positional-fallback.md
├── ...
├── 01-walking-skeleton-keyed-csvdiff.done.md       # handoff (stays in root)
├── 01-walking-skeleton-keyed-csvdiff.review.md     # review ledger (stays in root)
├── 01-walking-skeleton-keyed-csvdiff.state.json    # final state (stays in root)
└── done/
    └── 01-walking-skeleton-keyed-csvdiff.md        # archived task spec
```

The next task's `load` stage will:
- Pick up its own task .md from the root,
- Read every `*.done.md` in the root (one in the example above) for
  ground truth,
- Verify dependencies by checking that the dependent task ids each
  have a corresponding `.done.md`.
