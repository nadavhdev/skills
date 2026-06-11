# Workflow — stage-by-stage playbook

This is the detailed playbook the SKILL.md table refers to. It assumes
you have already (a) read the project's convention documents, and (b)
verified that a valid task file exists on disk. If you haven't, go
back to SKILL.md and do those gates first.

Every stage reads `state.json` on entry, runs its work, and writes
`state.json` on exit. The format is defined in `references/state-tracking.md`.

---

## Stage 1 — `load`

**Goal:** turn the user's reference into an absolute path on disk, read
everything you need to plan the work, and verify dependencies are done.

**Steps:**

1. Resolve the user's task reference to a file path. Acceptable forms:
   - Full path: `./tasks-csvdiff/01-walking-skeleton-keyed-csvdiff.md`
   - Numbered shorthand: `task-01` or `01` — resolves against the named
     `./tasks-<slug>/` directory if it's unambiguous in the working dir.
     If multiple `./tasks-*/` directories exist, ask which one.
   - Slug shorthand: `01-walking-skeleton` — fuzzy-match by filename
     prefix within the chosen tasks dir.
2. If you cannot uniquely resolve the reference, stop and ask the user.
   Do not guess between two candidate files.
3. Read the resolved task `.md` **end to end**. The acceptance criteria
   are the contract for everything that follows.
4. Read the tasks dir's `README.md` if present — it lists the dependency
   graph **and** the `Source TDD:` pointer at the top. Capture both.
   Verify every task this one depends on has a corresponding
   `*.done.md` in the tasks dir. If a dependency is not done, stop and
   tell the user: "task-XX depends on task-YY which is not yet done.
   Implement task-YY first, or override if you're sure."
5. **Locate and read the parent TDD.** The task spec exists only because
   the TDD does — and the spec has a "TDD sections addressed" line
   (e.g. `§0 Design constraints, §3 High-level approach, §4a Command
   surface...`) that lists exactly which TDD sections it implements.
   You must read those sections.
   - Resolve the TDD path: the `Source TDD:` line in the tasks dir
     `README.md` (e.g. `../TDD-csvdiff.md`); or the convention path
     `./TDD-<slug>.md` in the working dir where `<slug>` matches the
     tasks dir slug; or ask the user if neither resolves.
   - **Refuse to proceed if the TDD cannot be found.** A task spec
     without its parent TDD is decontextualized — you would be
     implementing a decomposition without the design it decomposes.
     Say so and stop.
   - Read the TDD end-to-end on first encounter; on later tasks in the
     same TDD, re-read only the sections this task's "TDD sections
     addressed" line cites, plus §0 (design constraints) and any open
     questions.
   - Record `tdd_path` and `tdd_sections_read` in `state.json`.
6. Read **every `*.done.md` in the tasks dir** (both the root and
   `done/`). These are the ground truth for what has already been built
   — decisions, file locations, conventions established by prior tasks.
   Do not skip this. The whole point of the handoff doc is to be read by
   the next task.
7. Re-read the relevant `CLAUDE.md` sections for any specifics the task
   touches (e.g. registration files, test layout, lint rules).
8. Check for an existing `state.json` for this task:
   - If none exists, create one with `current_stage: load`, `started_at:
     <now>`, `attempts: { load: 1 }`.
   - If one exists and `current_stage: done`, refuse: the task is
     already done; the handoff is at `task-XX.done.md`.
   - If one exists and `current_stage` is mid-pipeline, this is a
     resume. Tell the user explicitly which stage you're resuming from
     and what was last attempted. Then jump to that stage instead of
     restarting.

**Exit:** `current_stage: implement`. State file now holds the resolved
task path, the resolved TDD path and sections read, the list of
dependency done.md files read, and a timestamp.

**Failure modes you must guard:**

- Two tasks dirs in the working directory and an ambiguous reference →
  ask.
- Reference resolves to a file that doesn't exist → tell the user; do
  not "find a close match" silently.
- A dependency is missing its done.md → stop; do not assume it's done.
- **The parent TDD cannot be found** → stop; do not implement against
  the task spec alone. Ask the user for the TDD path or to author one
  via the tech-lead TDD-authoring capability.

---

## Stage 2 — `branch`

**Goal:** decide which branch the implementation lands on, switch the
working tree to it, and refuse to proceed if dependencies make a clean
branch impossible.

This stage exists **before** `implement` so that every file the
implementer writes lands on the correct branch from the start. The
old "create a branch at commit time if you're on master" approach
landed work on the wrong branch when dependencies forced staying on
a feature branch.

**Steps:**

1. **Detect the project's canonical base branch** from origin:
   ```bash
   base=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null \
            | sed 's@^origin/@@')
   ```
   Fall back to `master`, then `main`, in that order. If neither
   exists locally or in origin, ask the user.

2. **Classify each dependency's commit state.** For each task id in
   the task spec's `Depends on:` line (skip if "none"):
   - Find the dependency's commit sha. Try `state.json.commit_sha`
     first; fall back to the `**Commit:**` field in the dependency's
     `task-YY.done.md`. If neither resolves, **block** with:
     > "Dependency task-YY has done.md but I cannot find its
     > commit sha. The done.md may be a manual record without a
     > real commit. Cannot verify dependency state."
   - Test reachability from base:
     ```bash
     git merge-base --is-ancestor <dep_sha> <base>
     ```
   - If yes → `dep_state[dep] = "merged"`.
   - If no → `dep_state[dep] = "unmerged"`; capture which branches
     contain it: `git branch -a --contains <dep_sha>`.

3. **Decide branch strategy** by case:

   **Case A — No dependencies, OR every dep merged into base:**
   - Ask the user (per their preference):
     > "task-XX has no unmerged dependencies. Branch off:
     > (a) `<base>` (project default)
     > (b) the current branch `<current>`
     > Which would you like?"
   - With the chosen parent, create a new branch:
     ```bash
     git checkout -b <type>/<task-slug> <chosen_parent>
     ```
     - **`<type>` inference:** `feat` by default; `fix` if the task
       title mentions "fix" or "bug"; `docs` if it is docs-only
       (changelog / README / .rst); `chore` if it is build/CI/lint.
     - **`<task-slug>`:** derived from the task title or filename.
       Keep ≤30 chars; combine the feature slug (from the tasks-dir
       name `tasks-<feature>`) with a short summary from the title.
       Example: tasks-csvdiff/01-walking-skeleton… → `feat/csvdiff-walking-skeleton`.

   **Case B — Some deps unmerged, AND every unmerged dep's commit is
   reachable from the current branch:**
   - The dep code must stay accessible, so the new branch must be
     created **off the current branch** (not off base). This preserves
     the dep commits in the working tree while giving task-XX its own
     branch for a clean PR diff.
   - Create the new branch off `<current>`:
     ```bash
     git checkout -b <type>/<task-slug> <current>
     ```
     (Same naming rules as Case A — `feat` by default, derived from the
     task slug.)
   - **Then confirm with the user:**
     > "task-XX depends on task-YY (and Z), whose commits are unmerged
     > and reachable only from the current branch `<current>`. I'm
     > creating `<new-branch>` off `<current>` so the dep code is
     > available, and the PR will target `<current>`. Options:
     > (a) Proceed — create `<new-branch>` off `<current>` (my plan)
     > (b) Stop — I'll merge task-YY's branch into `<base>` first,
     > then re-invoke (PR will target `<base>` instead)
     > Which would you like?"
   - If (a): switch to the new branch. Record strategy as `"new"` and
     `branch_base: <current>` in state.json. The PR stage will target
     `<current>` (not `<base>`).
   - If (b): exit gracefully with a one-line "stopping per your
     instruction; merge task-YY's branch into `<base>` and re-invoke."

   **Case C — Some deps unmerged, AND at least one is NOT reachable
   from the current branch:**
   - **Block.** Surface to the user with the exact list of branches
     containing each unreachable dep:
     > "task-YY's commit `<sha>` is on branch(es) `<list>`, none of
     > which is the current branch `<current>`. To proceed:
     > (a) merge `<list>` into `<base>` and re-invoke me;
     > (b) switch to a branch that contains task-YY's commits and
     > re-invoke me;
     > (c) override (escape hatch): re-invoke with `--force-branch
     > <name>` to skip this check.
     > Which would you like?"
   - Do NOT auto-switch branches. That's a stateful operation that
     could clobber uncommitted work in the user's tree.

   **Case D — Dep commit is unreachable from ANY branch:**
   - **Block.** The dep's branch was probably deleted:
     > "Dependency task-YY's commit `<sha>` is not reachable from
     > any branch in this repo. The branch that held it may have
     > been deleted. Cannot proceed safely. Please restore the
     > branch or confirm task-YY needs to be redone."

4. **Record the decision** in `state.json`:
   ```json
   {
     "branch_strategy": "new" | "continue",
     "branch_name": "<branch>",
     "branch_base": "<base>",        // only when strategy = "new"
     "branch_decided_at": "<ts>"
   }
   ```

5. **Ask the pre-PR review-gate preference for this session** (once,
   interactive mode only). The review loop produces a ledger every task;
   this preference decides only whether to *pause* for your inspection
   before the PR is opened:
   - **Always** — pause before every PR.
   - **Conditional** (default) — pause only when the review was
     non-trivial (exact trigger defined in the `checkpoint` stage).
   - **Never** — never pause; just surface the ledger path at loop end.

   Record it as `review_gate_preference` in `state.json`. **In
   non-interactive mode, skip the question and store the default
   `"conditional"`** — the `checkpoint` stage never pauses without a
   human present, but keeping the real default (rather than forcing
   `"never"`) lets it still compute and record whether the gate *would*
   have fired, for the audit trail. Bundling this with the branch
   question keeps it from becoming a separate interruption point.

6. **Verify the working tree is clean** before continuing. If
   uncommitted changes exist outside the task's expected file set,
   stop and ask the user — don't risk discarding their work with a
   branch switch.

**Exit:** `current_stage: implement`. Working tree is on the chosen
branch; state.json records which and why.

**Failure modes:**

- Dep done.md missing a commit sha → block (Case D-adjacent).
- User has uncommitted changes that would conflict with the new branch
  → stop and ask.
- User explicitly invokes with `--force-branch <name>` → honor that
  branch and skip the strategy decision; still record the override in
  state.json.

---

## Stage 3 — `implement`

**Goal:** write the code the task asks for, mirroring project
conventions. You may write tests alongside the code as you go — a
function and its test, a service and its test, a component and its
test — if that's how you naturally work. The `tests` stage that
follows is a coverage-completion check, not a forbidden zone before
it. The only hard rule: by the time `tests` exits, every acceptance
criterion has a test.

**Steps:**

1. Re-read the task spec's "Composes" section and acceptance criteria.
   For each bullet, identify the file(s) it touches. Build a mental list
   of the exact files you will create or edit.
2. **Cross-reference each acceptance criterion to the TDD section it
   derives from.** The task spec is a decomposition of the TDD; every
   non-trivial behavior the spec lists should map back to a TDD section
   cited on the task's "TDD sections addressed" line. For each criterion:
   - Identify the TDD section it derives from. (For most tasks
     produced by the tech-lead breakdown capability, the mapping is
     direct: the "TDD sections addressed" line names the sections, and
     the criteria are organized in the same order.)
   - Confirm the criterion is consistent with what the TDD section
     actually says. If the spec restates the TDD imprecisely, **the
     TDD wins** — surface the discrepancy to the user before
     implementing (per the prime directive); do not silently
     "correct" the spec.
   - Record `tdd_correlation` in `state.json` as `{criterion → tdd_section}`
     so the review stage can audit it. An acceptance criterion that has
     no TDD anchor is a yellow flag — either the spec drifted or the
     TDD is incomplete; surface it.
3. Confirm there's nothing in the spec you don't understand. If there
   is, stop and ask before writing code.
4. Read the nearest neighbor file in the codebase if you haven't yet
   (e.g. for csvkit, read `csvjoin.py` if you're building a new
   utility). Mirror its structure exactly.
5. **Surface-before-deviating check.** Per the prime directive in
   SKILL.md, before writing code, scan the spec for any of:
   - A difficulty you can't resolve while honoring the spec literally.
   - A better way that you genuinely believe would improve the outcome.
   - An issue that may arise from the literal spec (perf, security,
     downstream breakage).
   If any of these are present **and you cannot disprove your own
   concern by simulating the spec's behavior against the relevant
   files** — stop and surface to the user with the script in SKILL.md's
   "Prime directive" section. Do not "improve" silently. Do not defer
   the surface to the handoff doc after shipping. The user picks the
   path; you implement what they pick. Log the surface + resolution as
   a decision in the handoff doc at the end.
6. Implement. Stay strictly inside the task's scope — do not refactor
   adjacent code, do not add hypothetical flexibility, do not "clean up"
   while you're there. Scope discipline is staff-level behavior.
7. After each non-trivial change, mentally check **two** things:
   (a) does this change match what the acceptance criteria asked for,
   or is it slightly different? (b) is the change still consistent with
   the TDD section the criterion derives from, or has it drifted from
   the design intent? If either is drifting, course-correct immediately.
8. **Persona self-check — required before exiting `implement`.** Walk
   through this checklist deliberately. For each item, answer
   yes/no/N-A out loud (i.e. in state.json's `self_check`) — do not
   skip any item. The reviewer will check these too, but catching them
   here saves a review round.

   Code quality:
   - **Clean names:** every public function / variable / file reads
     like prose? No `helper2`, `tmp_data`, single-letter non-iterator
     vars?
   - **Single responsibility:** every new function does one thing,
     stated in its name?
   - **SOLID seams:** dependencies point inward to abstractions, not
     outward to concretions? Anything obviously violating SRP?
   - **Decoupling:** no module reaches into another's internals? No
     hidden coupling via shared mutable state?
   - **Reuse:** no copy-pasted block where a helper exists or would
     be a 5-line addition?
   - **Readability:** can a peer reading this cold a year from now
     follow it without git-blaming?
   - **Extendability:** can the next dependent task build on this
     without rewriting?
   - **Maintainability:** is every bit of complexity justified by the
     spec? Anything speculative for "future flexibility" the spec
     didn't ask for? (If yes — remove it; per scope discipline, that's
     freelancing.)

   Correctness & safety:
   - **Security at boundaries:** every external input (stdin / network
     / env / file) validated at the first boundary? No injection vectors
     in any shell exec or query construction?
   - **Error handling:** every raise at the right layer with the right
     context? No silent except: pass? Exit codes match the spec exactly?
   - **No PII in logs:** any log statement emitting user data, tokens,
     credentials, or stack frames containing sensitive values?
   - **Scope discipline:** is the diff strictly inside the task's
     stated surface? Any "while I'm here" edits to unrelated files?

   If any item is "no" and you cannot defend the gap by spec, fix it
   before advancing. If you defer something deliberately (e.g. a
   refactor that needs broader scope), log it in `self_check.deferred`
   with a one-line reason; the handoff doc will surface it.

**Exit:** `current_stage: tests`. State file lists the files changed in
this stage, plus any pre-implementation surfaces and their resolutions
(`surfaces_raised: [{question, options, user_choice, ts}]`), plus the
`self_check` results (`{item, status, note}[]` plus optional
`deferred[]`).

**Things NOT to do here:**

- Don't run the *full* project test suite yet — that's the `verify`
  stage. (You may run individual tests for the code you just wrote to
  smoke-check it; the verify stage is what guarantees the full suite
  is green.)
- Don't try to skip ahead to commit. The review stage exists for a
  reason.
- Don't silently deviate from the spec because you think you have a
  better approach. The prime directive in SKILL.md is non-negotiable:
  surface first, deviate only after the user agrees.

---

## Stage 4 — `tests`

**Goal:** **coverage-completion check.** By the time this stage exits,
every acceptance criterion in the task spec has at least one test that
fails when the behavior breaks. If you wrote tests during `implement`
(interleaved with the code), most of this stage is reconciliation; if
you didn't, this is where you write them all. Either path is valid —
what's not valid is leaving acceptance criteria untested.

**Steps:**

1. Find the test file location and framework from project conventions
   (CLAUDE.md typically says explicitly; csvkit uses `unittest` classes
   in `tests/test_utilities/test_<name>.py`, with helpers from
   `tests/utils.py`).
2. Walk the acceptance criteria one bullet at a time. For each bullet:
   - If a test already exists for it (from interleaved work during
     `implement`), confirm it actually fails when the behavior breaks
     — read it; do not just count it.
   - If no test exists, add one.
3. Cover error paths the spec calls out: exit codes, parse failures,
   user-input errors. Do not skip these because "they're hard to test."
   They're the most common review finding.
4. Add fixtures to wherever the project keeps them (csvkit:
   `examples/`). Reuse existing fixtures where they fit; do not create
   redundant ones.
5. Include any project-convention-mandated tests
   (e.g. csvkit requires `test_launch_new_instance` for every utility).

**Exit:** `current_stage: verify`. State file lists test files added /
modified, and confirms every acceptance criterion is mapped to at
least one test.

---

## Stage 5 — `verify`

**Goal:** run the project's test and lint commands and get them all
green. This stage is a tight loop until clean.

**Steps:**

1. Source the commands from the project's convention docs. For csvkit
   (per `.claude/CLAUDE.md`), this is:
   - `pytest --cov csvkit`
   - `flake8 .`
   - `isort . --check-only`
   - `check-manifest`
   Use the exact commands the project documents. Do not invent your own.
2. **Classify each command as hard-gate or substitutable.** Most are
   hard-gate (tests, lint) — they must pass for verify to succeed.
   Some are *substitutable* — they verify a property the project cares
   about, but if the environment blocks them (corp network proxy, no
   internet, missing system tool), you may substitute a documented
   manual inspection. The classic example is `check-manifest`, which
   requires PyPI access to bootstrap setuptools for its build step; in
   a blocked-network environment, manually verifying that `MANIFEST.in`
   covers all new files is an acceptable substitute. Treat a command
   as substitutable only when (a) the project conventions don't say
   it's a hard gate, and (b) you can do an equally rigorous manual
   check, and (c) you document the substitution in `state.json` with
   the reason and the manual check performed. **Do not declare
   substitutable to avoid work — only when the environment prevents
   the automated check.**
3. Run them. If any hard-gate command fails, fix the underlying issue.
   Then re-run **all** hard-gate commands, not just the one that
   failed (a fix in one place often breaks something else).
4. Loop until all hard-gate commands pass (and any substitutable ones
   are either passed or documented as substituted with manual check).
5. If a hook or CI command requires bypass (`--no-verify`,
   `# noqa: E501`, `# type: ignore`, etc.), do **not** bypass. Fix the
   underlying issue. The one exception is when the bypass is itself an
   established project convention documented somewhere — and then
   reference where it's documented.
6. Update `state.json.attempts.verify` each iteration so you can see in
   the state how many fix cycles it took.

**Exit:** `current_stage: review`. State file records the commands run
and their final-green status.

**Failure mode:** you cannot make a test pass. Stop and ask the user.
Do not delete the failing test, comment it out, or weaken the
assertion. Those are all worse than asking.

---

## Stage 6 — `review`

**Goal:** get a fresh tech-lead-perspective review of your work,
incorporate the findings, and pass review.

**See `references/review-loop.md` for the full protocol.** High-level:

1. Spawn a fresh subagent. Tell it to read the tech-lead skill's
   implementation-review capability (`~/.claude/skills/tech-lead/
   references/capability-review.md`) and run it against the diff.
2. Pass the subagent:
   - The path to the task spec file.
   - The current branch's diff against the merge base of `master` (or
     whichever main branch the project uses).
   - The paths of all sibling `*.done.md` files for context.
   - The project's `CLAUDE.md` path if relevant.
   - **The review mode for this round.** Round 1 is a FULL review;
     rounds ≥ 2 are TARGETED re-reviews scoped to the fixes — pass the
     review ledger (`task-XX.review.md`) and the list of files/hunks you
     changed since the last round. See `review-loop.md` → "Scoping the
     review by round."
3. The subagent returns a structured verdict: `APPROVE` or
   `REQUEST_CHANGES` plus a list of findings, each with severity
   (`blocker`, `major`, `minor`, `nit`) and an actionable description.
   On rounds ≥ 2 it also returns `prior_findings_review` — a closure
   status (`closed` / `not_closed` / `reopened`) for each prior finding.
4. **Each round, append a section to the review ledger**
   `task-XX.review.md`: the findings with their code evidence when the
   verdict arrives, then the Resolution + Fix blocks once you act on
   them. The ledger is the human audit trail and the input that lets the
   next round scope down. See `review-loop.md` → "Writing the review
   ledger."
4. If `APPROVE`:
   - If the round had **zero findings**, the loop has converged — exit
     to the `checkpoint` stage (Stage 6b), which decides whether to
     pause for the user before `commit`.
   - If the round had `minor` or `nit` findings and you applied any of
     them, the diff that was reviewed is now stale. Re-run `verify`,
     then **spawn one more review round** to confirm your fixes are
     clean and didn't introduce regressions. This is the
     post-APPROVE re-review: it costs one extra round but prevents
     committing un-reviewed changes. Skip it only if you deferred
     every finding to follow-up (in which case nothing changed since
     the review).
5. If `REQUEST_CHANGES`:
   - Address every `blocker` and `major` finding. They are not
     negotiable.
   - Address `minor` findings unless they conflict with the task spec.
   - You may defer `nit` findings to a follow-up — but log the
     deferred nits in the handoff doc.
   - Re-run the `verify` stage (tests + lint must still be green after
     your fixes).
   - Re-invoke a fresh review subagent.
6. **Hard cap: 3 review rounds.** If round 3 still returns
   `REQUEST_CHANGES` with `blocker` or `major` findings, stop and
   surface the situation to the user — something fundamental is wrong
   (spec ambiguity, conflicting constraints, the wrong design). Do not
   keep iterating in the hope it converges. (Note: post-APPROVE
   re-reviews count toward the cap, but in practice rounds 1 and 2
   resolve cleanly when round 1 was APPROVE.)

**Exit:** `current_stage: checkpoint`. State file records all review
rounds (input diff sha, verdict, findings, prior-finding statuses, fixes
applied), and `task-XX.review.md` holds the human-readable ledger with
code evidence and fix diffs for every round.

---

## Stage 6b — `checkpoint` (pre-PR review gate)

**Goal:** give the user a chance to inspect the review-fix history and
ask for final changes before the PR is opened — without ever letting
unreviewed code through.

The review loop, left alone, runs straight from APPROVE into commit/PR.
For a non-trivial review (multiple rounds, real severity) the user may
want to read the ledger, gauge the complexity, and tune before shipping.
For a clean review, stopping would just be friction. This gate's whole
job is to tell those two apart — and, when it does stop, to make sure
any tuning is itself reviewed.

**Steps:**

1. **Surface the ledger path — always, in every mode.** Print the path
   to `task-XX.review.md` plus a one-line complexity summary: round
   count, the max severity addressed, and any deferred nits. This is the
   cheap, always-on half of observability; it happens whether or not the
   gate pauses.

2. **Evaluate the gate** from `state.json.review_gate_preference`:
   - `never` → do not pause; advance to `commit`.
   - `always` → pause (step 3).
   - `conditional` (default) → pause **iff the loop took more than one
     round to converge AND at least one round raised a `major` or
     `blocker`.** Otherwise advance to `commit`. A clean one-round
     APPROVE, or a multi-round loop that only churned on minors/nits,
     does **not** pause. (A `major`/`blocker` always forces a
     REQUEST_CHANGES → fix → re-review, so it inherently implies > 1
     round — the two signals reinforce rather than fight.)
   - **Non-interactive mode** (no human to answer) → never pause,
     regardless of preference. Record in the handoff whether the gate
     *would* have fired (rounds + max severity + ledger path) so it is
     auditable after the fact. Advance to `commit`.

3. **Pause and hand control to the user.** Show the ledger path, the
   complexity summary, and a one-line invitation: proceed, or raise
   concerns / more asks. Then wait. The user picks:
   - **Proceed** → advance to `commit`.
   - **Raise concerns / request changes** → implement the requested
     changes, then **run them through the review-fix loop** (Stage 6 —
     a targeted re-review + ledger append) until they re-converge to
     APPROVE. Then **return to step 1 of this stage** and re-surface the
     updated ledger. Repeat until the user proceeds.

   **Hard invariant — the checkpoint is not a bypass around review.**
   No code reaches the PR without passing review. Every change made
   during the pause re-enters the review loop (see `review-loop.md` →
   "Re-entering the loop from the pre-PR checkpoint"). The checkpoint is
   for final checks and tuning, not for sneaking unreviewed edits in
   ahead of the PR.

**Exit:** `current_stage: commit`. State file records the gate decision
(`fired` / `passed-through` / `skipped-non-interactive`); any changes
the user requested during the pause are recorded as additional review
rounds in `review_rounds` and the ledger, exactly like any other round.

**Failure modes you must guard:**

- You implement a pause-driven change but skip re-review → **invariant
  violation.** Always route pause-driven edits back through Stage 6
  before proceeding to `commit`.
- A non-interactive run tries to pause → never pause without a human;
  behave as `never` and record the would-have-fired flag in the handoff.

---

## Stage 7 — `commit`

**Goal:** stage the right files, write a good commit message, commit.

**Steps:**

1. **Confirm the branch is the one Stage 2 decided.** Read
   `state.json.branch_name` and `state.json.branch_strategy`. Compare
   to the current branch (`git branch --show-current`). They must
   match. If they don't, something has gone wrong (a hook switched
   branches, or the user manually checked something out mid-pipeline);
   stop and surface to the user rather than committing on the wrong
   branch. **Do not create a new branch in this stage** — that
   decision was made and recorded in Stage 2.
2. **Stage files by name**, not with `git add -A`. The state file
   already lists the files touched in `implement` and `tests` stages —
   use that list. Sanity-check with `git status` that nothing
   unexpected is present.
3. Write a commit message that:
   - Matches the project's recent commit style (check `git log
     --oneline -20`).
   - Has a short subject line referencing the task title (under ~70
     chars).
   - Has a body explaining the *why*, not just the *what*. The diff
     shows the what.
   - References the task file path so the link is reviewable.
4. **Never use `--no-verify`.** If pre-commit hooks fail, the commit
   did not happen — go back to `verify`, fix, and try again with a NEW
   commit (do not amend).
5. Run `git status` afterward to confirm the commit landed.

**Exit:** `current_stage: pr`. State file records the commit sha,
branch name, and files committed.

---

## Stage 8 — `pr`

**Goal:** push the branch and open a PR with the **exact body
structure** below — no improvisation.

**Steps:**

1. Push the branch to the remote with `-u` if it's a new branch.
2. Open a PR via `gh pr create` using a HEREDOC for the body to
   preserve formatting.
   - **Title:** derived from the task title, under ~70 chars. Prefix
     with conventional-commit-style tag if the project uses them
     (`feat:`, `fix:`, etc.).
   - **Body:** follow the canonical structure in the next subsection
     exactly. Do not add sections; do not omit sections; do not
     reorder.
3. Capture the PR URL.

**Exit:** `current_stage: handoff`. State file records the PR URL.

**Note:** PR creation is a shared-state action. If the user has not
already authorized PR creation for this session, confirm before
pushing.

### Canonical PR body structure (mandatory — five sections, in order)

The PR body has exactly **five sections**, in this order. Every
dev-expert PR uses this structure. The reviewer audits the PR body
against this template; deviations are a `docs` finding (severity by
how much the deviation harms reviewability).

```markdown
## Ticket
<reference to the task file — if local, the workspace-relative path,
ideally as a markdown link: `[tasks-csvdiff/03-no-key-positional-fallback.md](tasks-csvdiff/03-no-key-positional-fallback.md)`.
If the source was a Jira / Trello ticket, the ticket URL or ID. One
line, no prose around it.>

## Description
<2–3 line high-level description of what this PR is for. Plain
English, focused on the "why" and the user-visible outcome. Not a
restatement of the task title; not a list of files; not a status
update. Stop at 3 lines.>

## Key changes
<short, one-line key bullets that name the key functionality this PR
delivers. Each bullet is one concrete capability the user or
downstream code now has. No implementation detail unless it changes
behavior. 3–7 bullets typically; do not pad.>

- <one-line key bullet>
- <one-line key bullet>
- ...

## Files
<table of the key source files touched by this PR. Test files are
NOT listed here — they go in the Testing section as behavioral
coverage, not as file inventory. Documentation files (rst, md,
CHANGELOG) belong here if the task explicitly shipped them.

**Collapse rule for trivial sets:** when you add a group of similar
files that exist to support one concept (e.g. a set of CSV fixtures
for `--ignore` testing, a set of icon assets at multiple sizes), list
them as a SINGLE row with all paths in the File column and one shared
Purpose. Padding the table with one row per fixture is visual noise
that hides the load-bearing files.>

| File | Change | Purpose |
|---|---|---|
| [`path/to/file.py`](path/to/file.py) | added/modified/deleted | <one-line purpose> |
| [`path/to/other.py`](path/to/other.py) | added/modified/deleted | <one-line purpose> |
| [`examples/diff_a.csv`](examples/diff_a.csv), [`examples/diff_b.csv`](examples/diff_b.csv) | added | Canonical changed/added/removed fixture pair |

## Testing
<key testing coverage, as short concise bullets. Each bullet names a
behavior under test, not a test name or test count. **Do not include
test stats. Do not include "added N tests." Do not list test file
paths.** This section is about coverage shape, not test inventory.>

- <behavior under test>
- <behavior under test>
- ...
```

#### Rules the reviewer enforces against this structure

- **Ticket section:** must reference a real path or URL. A missing or
  broken reference is at least a `docs` `major`.
- **Description:** must be 2–3 lines. A wall-of-text description that
  goes past 3 lines is a `docs` `minor`. A description that's just a
  paraphrase of the title is a `docs` `minor`.
- **Key changes:** each bullet must name a capability, not a code
  change. "Added `_parse_key()` helper" is a code change (wrong shape);
  "supports composite keys in `-c`" is a capability (correct shape).
- **Files table:** must not include test files. Must use markdown
  link syntax for the file column. Must include the Change column
  with one of `added`/`modified`/`deleted`. A Purpose column entry
  longer than a sentence is a `docs` `nit`.
- **Testing:** must not include stats, counts, or test file paths. A
  bullet that reads "added 12 tests covering X" is wrong-shape; the
  right shape is "X behavior" (the coverage), full stop.

#### Why this structure

A consistent PR body shape means the reviewer (human or tech-lead
subagent) can find the load-bearing information in the same place
every time. The structure also enforces scope discipline — if you
cannot fill out "Key changes" in 3–7 capability bullets, the PR is
probably either too small (combine?) or too big (split?). The "list
behaviors, not counts" rule for the Testing section exists because
counts don't tell a reviewer what's covered — behaviors do.

---

## Stage 9 — `handoff`

**Goal:** write the `task-XX.done.md` handoff document, move the
original task file into `done/`, and mark state as `done`.

**Steps:**

1. Write `task-XX.done.md` in the tasks dir **root** (not in `done/`).
   It stays in the root so subsequent tasks can read it without having
   to scan the archive. Use the template in
   `references/state-tracking.md`. Its "Review findings & resolutions"
   section is a one-paragraph summary that **links to the ledger**
   (`task-XX.review.md`) rather than duplicating it.
2. Move the original `task-XX.md` from the tasks dir root into
   `./tasks-<slug>/done/`. Create `done/` if it doesn't exist.
3. Update `state.json`:
   - `current_stage: done`
   - `completed_at: <now>`
   - `pr_url`, `commit_sha`
   - Final review verdict
4. **Do not move `state.json` or `task-XX.review.md`** — both stay in
   the tasks dir root next to the done.md so the full trail (machine
   state + human ledger) is auditable.

**Exit:** workflow complete. Report to the user:
- The PR URL
- The path to the handoff doc
- Any deferred nits or unresolved findings worth flagging
