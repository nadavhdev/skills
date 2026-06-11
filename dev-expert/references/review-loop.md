# Review loop — how to run the tech-lead implementation review

The `review` stage spawns a fresh subagent that adopts the tech-lead
skill's **implementation-review** capability and returns a structured
verdict. You (dev-expert) consume that verdict and apply changes.

This separation is the whole point. **You are the implementer. You are
not the reviewer.** A reviewer with fresh context catches what you, with
your hands on the code, will rationalize as "already fine."

---

## When to surface to the user — the four triggers

The review loop is designed to converge automatically — implement,
review, fix, re-review, commit. But there are **four explicit
scenarios** where the loop stops and hands control back to the user
for a judgment call. This list is the single source of truth; the
individual scenario sections later in this file reference it. If any
of these fire, **do not push past them**.

| # | Trigger | When it fires | What to say |
|---|---|---|---|
| 1 | **Per-finding inability to apply or decide** | A `blocker` finding you cannot apply (don't know how, or the fix conflicts with the spec); a `major` finding whose fix would require deviating from the spec; *or* any finding where, after running Step A's simulation, you genuinely cannot decide whether it holds | "Round N flagged X. I can't apply it because Y / I cannot honestly decide whether it holds. Need a judgment call: (a) apply as flagged, (b) defer with reason, (c) revise the spec." |
| 2 | **Two-round persistent disagreement** | The same finding is raised in two consecutive rounds and you have an evidence-based rebuttal (per Step C) that the reviewer is not accepting | "Round N: review still flags `<finding>`. My rebuttal (see state.json round N-1) is `<one-liner>`. Reviewer not budging. Options: (a) apply the finding, (b) accept my rebuttal and proceed, (c) revise the spec / TDD." |
| 3 | **Three-round hard cap reached** | Round 3 returns `REQUEST_CHANGES` with any `blocker` or `major` finding. **Never invoke round 4 automatically.** | "Review round 3 still requests changes (blockers: `<n>`, majors: `<n>`). I'm stopping the auto-loop. Unresolved findings: `<list>`. Likely root cause: `<best guess>`. How would you like to proceed?" |
| 4 | **JSON contract failure** | The subagent returned non-JSON (or invalid JSON) twice in a row — once on the original call, again after the re-prompt | "Review subagent returned malformed JSON twice. I cannot programmatically parse the verdict. Either the model is misbehaving on this round or the contract needs revision. How would you like to proceed?" |

**Common shape of every surface:** state what happened, state what you
tried, state the options the user can pick from, then stop. Do not
recommend an option unless asked. Do not silently take any path
forward. Update `state.json` with the surface event before stopping
so any later resume sees it.

Detailed protocol for each trigger lives in the appropriate section
below (Fix loop Step D for trigger 1, Step E for trigger 2, "Round
cap" section for trigger 3, "Parsing the verdict" for trigger 4).

---

## When to invoke

Only from the `review` stage. Specifically:

- After `verify` has passed all tests + lint (so you're not asking the
  reviewer to read code that doesn't even pass tests).
- Before any commit / push / PR.
- Each round of the review loop spawns a **new** subagent. Do not
  send a follow-up message to the previous one. A fresh subagent each
  round keeps the reviewer from being primed by your prior responses —
  this is about *who* reviews, not *what* they see. On rounds ≥ 2 you
  still hand that fresh subagent scoped inputs (the review ledger + the
  incremental diff) so it can run a targeted re-review instead of a
  full from-scratch pass; see "Scoping the review by round" below.

---

## Scoping the review by round

The review loop converges faster when each round reviews what actually
changed since the last one — without losing the safety of catching a
regression that a fix introduced. Scope therefore depends on the round.

### Round 1 — full review

Pass the **full branch diff** (`git diff <merge-base>..HEAD`, plus the
contents of any untracked files). The reviewer runs the complete
capability: all 9 lenses + all 6 mandatory checks, from a blank slate.
This is the baseline review and it must be thorough — everything
downstream trusts it.

### Rounds ≥ 2 — targeted re-review

By round 2 the only things that changed are the fixes you applied for
the prior round's findings. Re-deriving every finding over the unchanged
code is wasted work. Instead, give the reviewer enough to run a
**targeted re-review** (the tech-lead capability has a matching mode —
see `~/.claude/skills/tech-lead/references/capability-review.md` →
"Targeted re-review mode (review rounds ≥ 2)"):

- **The review ledger** (`task-XX.review.md`) — the prior rounds'
  findings *and the fix applied to each*. This is what tells the
  reviewer what changed and why.
- **The incremental context** — the explicit list of files/hunks you
  changed since the last round. You already know these: they are the
  edits you made applying the findings. Name them in the prompt.
- **The full current diff** — still provided, for regression context.

The reviewer then does three things instead of a full re-walk:

1. **Verifies each prior finding is actually closed** — and reports a
   per-finding status (`closed` / `not_closed` / `reopened`).
2. **Deep-reviews the changed hunks** — full lens treatment on the new
   code, because a fix is new code that has never been reviewed.
3. **Regression-scans the rest** — a lighter pass asking only "did any
   of these fixes break something a previous round approved?"

It returns the same JSON verdict. A regression a fix introduced is a
**new finding**; a prior finding that isn't actually closed is
**reopened** at its original severity. APPROVE still requires zero
`blocker`/`major` across both new findings and reopened priors.

**Why keep the regression scan** (rather than reviewing *only* the
changed hunks): the original design re-reviewed everything every round
precisely because a fix can break approved code. Targeted re-review
keeps that safety as the lighter step-3 scan while dropping the
expensive part — re-deriving findings over code that didn't change. If
you ever need to drop the regression scan too, that's a deliberate
trade (faster, but a fix-induced regression can slip through); don't
make it silently.

## Writing the review ledger

Every round, append a section to `task-XX.review.md` (template and field
rules in `references/state-tracking.md`). Write it at **two moments** so
the ledger is never half a round behind:

1. **After the verdict comes back** — write the round header, verdict,
   reviewer summary, and each finding with its **code evidence** (copy
   the actual lines the finding points at, with a `# file:line`
   header). On rounds ≥ 2, also record the reviewer's per-prior-finding
   status list.
2. **After you apply / rebut / defer** — fill each finding's Resolution
   and **Fix** block: the fix as a real diff hunk, or `n/a — rebutted`
   (with the Challenge block carrying the evidence), or `n/a —
   deferred`.

The ledger is append-only and is written from the same data as
`state.json.review_rounds`, in the same step — it just adds the code
evidence and the fix diff that JSON can't hold. It is **not optional**:
it is both the human audit trail and the machine input that makes the
rounds-≥-2 targeted re-review possible. A loop with no ledger forces
every later round back into a full from-scratch review.

## How to spawn the review subagent

Two mechanisms are supported, in this order of preference:

### Preferred: the `Agent` tool

If you have the `Agent` tool available (you are running as the primary
session, or your harness exposes it), use it. Pass it the prompt from
the "Subagent prompt template" section below, with `subagent_type`
matching your harness's general-purpose agent type. This is the
cleanest path — direct tool, structured result, harness-managed
isolation.

### Fallback: `claude -p` subprocess

If the `Agent` tool is NOT available (most commonly because dev-expert
itself is running as a subagent and the harness does not provide
recursive Agent access), use the Claude Code CLI in print mode as a
subprocess. This is the canonical fallback. Recipe:

```bash
# 1. Write the diff to a file the prompt references
git diff <merge-base>..HEAD > /tmp/<task-slug>-review-round-<N>.diff
# (Plus full contents of untracked files if needed — concatenate to the diff.)

# 2. Write the full review prompt to a file (use the template below)
cat > /tmp/<task-slug>-review-round-<N>-prompt.txt <<'PROMPT_EOF'
<full subagent prompt — see "Subagent prompt template" below>
PROMPT_EOF

# 3. Invoke claude -p with the prompt; pipe stdin, capture JSON
cat /tmp/<task-slug>-review-round-<N>-prompt.txt | \
  claude -p \
    --output-format=json \
    --allowedTools=Read,Bash \
    > /tmp/<task-slug>-review-round-<N>.response.json 2>&1

# 4. Parse the JSON result. The verdict field is what you act on.
```

Notes on the `claude -p` recipe:

- `--output-format=json` is required — the response file is structured;
  the `result` field at the top level contains the subagent's final
  output (your raw JSON verdict).
- `--allowedTools=Read,Bash` is the minimum — the reviewer needs Read
  for files and Bash for `git diff` / `cat`. Do not add tools the
  reviewer doesn't need.
- The subprocess inherits the project's `CLAUDE.md` and any installed
  skills (because `--bare` is NOT set). The reviewer will read
  `~/.claude/skills/tech-lead/SKILL.md` + `capability-review.md` per
  the prompt instructions.
- 10 minutes is a generous timeout for a review round. Adjust if your
  diffs are unusually large.
- If the JSON output is malformed (rare, but possible), re-spawn once
  with a shorter prompt: "Your previous response wasn't valid JSON.
  Re-emit it as raw JSON matching the contract." If that also fails,
  surface to the user (trigger #4 from the consolidated table at the
  top of this file).

**Why this fallback exists:** dev-expert is designed to be invoked
both as a primary session and as a subagent (e.g. from an automated
pipeline or another orchestration skill). The `Agent` tool is
typically only available at the top level, so subagent-from-subagent
review needs a different mechanism. `claude -p` provides that with no
new infrastructure.

---

## What to pass the subagent

The subagent has no memory of this conversation. It needs everything it
will need to do a real principal-eng review, packaged into a single
prompt. Concretely:

| Input                            | Why it matters                                                              |
|----------------------------------|-----------------------------------------------------------------------------|
| Path to the task spec `.md`      | The contract the implementation must satisfy.                                |
| **Path to the parent TDD**       | The design the spec decomposes. The reviewer must be able to check that the implementation honors the TDD, not just the spec's restatement of it. Sourced from `state.json.tdd_path`. |
| **`tdd_correlation` map**        | The implementer's claim about which TDD section each criterion derives from. The reviewer audits this — drift from the cited TDD section is a finding. Sourced from `state.json.tdd_correlation`. |
| **`self_check` results**         | The implementer's pre-exit-implement-stage walkthrough of the persona checklist (clean code, SOLID, decoupling, reuse, readability, extendability, maintainability, security, error handling, no PII in logs, scope). The reviewer **audits this**: any item marked "yes" that the diff disproves is a finding (severity by area). Sourced from `state.json.self_check`. |
| Path to the project's `CLAUDE.md`| The conventions the implementation must honor.                               |
| Paths to all sibling `*.done.md` | The ground truth of what was already built — review needs to know what's downstream and what was decided. |
| The full current diff            | The full branch diff (`git diff <merge-base>..HEAD`, plus untracked file contents). Passed **every** round — on round 1 it is the review surface; on rounds ≥ 2 it is the regression-scan context. |
| **Review mode + round number**   | `round 1 → full review`; `round ≥ 2 → targeted re-review`. Tells the reviewer which scope to run (see "Scoping the review by round"). |
| **The review ledger** (rounds ≥ 2) | `task-XX.review.md` — the prior findings and the fix applied to each. The reviewer reads it to know what changed and to verify each prior finding is closed. |
| **The incremental change list** (rounds ≥ 2) | The explicit files/hunks you changed since the last round (the edits you made applying findings). Focuses the deep-review pass. |
| The verify commands' results     | Confirms lint + tests are green before the review begins.                    |
| Round number + prior findings    | On round ≥ 2, the reviewer verifies each is actually closed and reports a per-finding status; a prior finding not closed is reopened at its original severity. |
| Round number + prior rebuttals   | Evidence-based pushbacks from prior rounds, so the reviewer can re-examine — not to argue, but to see the evidence the implementer found. |

---

## Subagent prompt template

Use this template (or close to it) for the `Agent` tool's `prompt` field.
Adapt the bracketed parts; keep the structure.

```
You are running the tech-lead skill's implementation-review capability.

Read these files FIRST, in this order, before doing anything else:
1. ~/.claude/skills/tech-lead/SKILL.md
2. ~/.claude/skills/tech-lead/references/capability-review.md
3. <project-root>/.claude/CLAUDE.md  (if present)
4. The task spec: <absolute path to task-XX.md>
5. The parent TDD: <absolute path from state.json.tdd_path>. Focus on
   the sections cited by the task's "TDD sections addressed" line — but
   you may consult other sections if a finding needs broader context.
6. The handoff docs (read all of these for ground truth on what's
   already been built): <list of *.done.md paths>
7. ON ROUNDS ≥ 2 ONLY — the review ledger: <path to task-XX.review.md>.
   It holds every prior finding and the exact fix applied to it. Read it
   before the diff so you know what changed and what to verify.

Review mode: <"round 1 — FULL review" | "round <N> — TARGETED re-review">.
- FULL review: run the complete capability — all 9 lenses + all 6
  mandatory checks — over the whole diff, from a blank slate.
- TARGETED re-review: run the "Targeted re-review mode (review rounds
  ≥ 2)" workflow in capability-review.md. Verify each prior finding is
  closed (report a per-finding status), deep-review the changed hunks
  listed below, and regression-scan the rest. The files/hunks changed
  since the last round are: <inline list, or "see ledger round <N-1>
  Fix blocks">.

Then read the diff to review. The diff has been written to:
<path>/review-input-round-<N>.diff

Also consult the implementer's `tdd_correlation` map (criterion → TDD
section): <inline JSON from state.json.tdd_correlation>. One of your
jobs is to audit it — if the implementation drifts from the cited TDD
section, that is a finding (severity by impact); if a criterion has
no TDD anchor, that is a finding worth surfacing.

Also consult the implementer's `self_check` walkthrough (persona
checklist): <inline JSON from state.json.self_check>. Audit this
too — any item marked "yes" that the diff disproves is a finding (e.g.
self_check says "clean names: yes" but you find `tmp_data2` → file a
`code-quality` finding). Any item marked "deferred" must be either
justified in the deferral note or flagged.

It is the output of:
  git diff <merge-base-of-master>..HEAD -- <file paths>

Your job:
- Review the diff against the task spec, the project conventions, and
  the prior done.md decisions.
- Apply the severity model from capability-review.md (blocker / major /
  minor / nit).
- Return a structured verdict in EXACTLY this JSON format, with no
  prose before or after:

{
  "verdict": "APPROVE" | "REQUEST_CHANGES",
  "summary": "<2-4 sentence overall assessment>",
  "findings": [
    {
      "severity": "blocker" | "major" | "minor" | "nit",
      "area": "<one of: correctness, spec-compliance, conventions, testing, error-handling, performance, security, docs, scope, observability, api-design, concurrency, dependency, code-quality, maintainability>",
      "file": "<path or null if cross-cutting>",
      "line": <int or null>,
      "summary": "<one-line summary>",
      "detail": "<2-6 sentence detailed explanation incl. what to change and why>",
      "spec_anchor": "<which acceptance criterion or done.md decision this connects to, or null if it's a general principal-eng observation>"
    }
  ],
  "prior_findings_review": [
    {
      "ref": "<prior finding id, e.g. '1.1'>",
      "status": "closed" | "not_closed" | "reopened",
      "note": "<one line: how the fix closed it, or why it didn't>"
    }
  ]
}

(Omit `prior_findings_review` entirely on a FULL review / round 1. On a
TARGETED re-review it is REQUIRED and must cover every prior finding. A
`not_closed` or `reopened` entry must also appear as a finding in
`findings` at its original severity.)

Context:
- This is review round <N> of at most 3.
- Prior rounds' findings: <inline JSON of prior findings or "none">. On
  a FULL review (round 1 this is "none"). On a TARGETED re-review,
  verify each prior finding is actually closed by the fix recorded in
  the ledger, and report a per-finding status in `prior_findings_review`
  (closed / not_closed / reopened). A prior finding that isn't closed is
  reopened at its original severity.
- Prior rounds' rebuttals from the implementer (evidence-based pushbacks
  to re-examine, not to argue with): <inline JSON of prior rebuttals or
  "none">.
- Verify commands all PASSED prior to this review:
  <list of verify commands run>. Trust these results — you are
  strictly read-only and do not re-run them.

Reminders from capability-review.md (apply every round):

- Lens walkthrough — walk EACH of the 9 lenses deliberately, in order,
  emitting findings as you go. Coverage discipline matters more than
  speed; skipping a lens is the most common way reviews miss issues.
  Primary lenses (in priority order):
    1. Spec compliance — every acceptance criterion delivered?
    2. TDD alignment — every cited TDD section honored?
    3. Project conventions — CLAUDE.md MUST/SHOULD/MAY classification
    4. Prior decisions — every sibling done.md decision respected?
    5. Code quality & maintainability — clean code, SOLID, decoupling,
       reuse, readability, extendability, maintainability
  Cross-cutting risk lenses (apply to every diff, even if it seems
  "not about" them):
    6. Dependency safety — new runtime deps justified?
    7. Observability & instrumentation — logs/metrics/traces present,
       no PII, appropriate log levels
    8. API / contract stability — any unannounced breaking change?
    9. Data migration safety — schema/backfill risk (skip if N/A)

- Hard-coded severity rules — apply without judgment:
  * Missing test for a spec-listed error path = blocker
  * Any security finding = blocker
  * Unannounced public-API breaking change = blocker
  * CLAUDE.md MUST violation = blocker, SHOULD = major, MAY = minor

- Mandatory checks — run all six every round (these come AFTER the
  lens walkthrough, not instead of it):
  * Trace every new public surface end-to-end
  * Verify every acceptance criterion has at least one test
  * Audit the tdd_correlation map against the cited TDD sections
  * Check every external input boundary for validation
  * Code-quality pass: clean code, SOLID, decoupling, reuse,
    readability, extendability, maintainability
  * Forward-looking bug hunt: what could go wrong that the test
    suite doesn't catch (boundary inputs, resource/state leaks,
    silent type coercions, retry-without-idempotency, etc.)

Do NOT:
- Write code or edit files. You are reviewing only and strictly
  read-only — no test runs, no linters, no scanners.
- Suggest commits or PR text.
- Approve if there is any blocker or major issue. APPROVE means "this is
  ready to commit as-is, after any nits are either fixed or deferred."
- File a finding just to "show" you ran a mandatory check. Clean checks
  produce no findings.
- Wrap the JSON in markdown code fences or any other formatting. Return
  raw JSON only.
```

### Why a JSON return contract

You need to parse the verdict programmatically (loop on REQUEST_CHANGES,
record findings in state.json, decide which to address). Free-form
review prose forces you to do error-prone NLP on every round. Structured
JSON makes the loop deterministic. The tech-lead review capability
mirrors this contract — both sides agree on the shape.

---

## Parsing the verdict

1. Parse the subagent's response as JSON. If it isn't valid JSON,
   re-spawn the subagent **once** with a shorter prompt that just says
   "Your previous response wasn't valid JSON. Re-emit it as raw JSON
   matching the contract." If that also fails, surface to the user.
2. Append a `review_rounds[i]` entry to `state.json` with:
   - `round`: i (1-indexed)
   - `diff_sha`: short sha of the diff input
   - `verdict`: from JSON
   - `findings`: from JSON
   - `prior_findings_review`: from JSON (rounds ≥ 2 only)
   - `ts`: now
3. **Write the ledger's round section now** (first of its two writes):
   round header, verdict, reviewer `summary`, every finding with its
   code evidence, and — on rounds ≥ 2 — the `prior_findings_review`
   status list. See "Writing the review ledger" and the template in
   `references/state-tracking.md`. (The Resolution + Fix blocks get
   filled in after you act on the findings.)
4. If `verdict == "APPROVE"` with no findings to apply: the review loop
   has converged — advance to the `checkpoint` stage. (The ledger's
   Outcome block is written when the checkpoint clears to `commit`, not
   here, because a pause may add more rounds.)
5. If `verdict == "REQUEST_CHANGES"` (or APPROVE with minors/nits you'll
   apply): enter the fix loop below.

---

## Fix loop — challenge every finding before you apply it

You are a staff-level engineer. You do **not** apply findings reflexively;
you also do not argue for the sake of arguing. The discipline is:
**challenge every finding with critical eyes, backed by evidence, then
decide whether it holds.** Most findings will hold. The ones that don't
matter just as much.

For **every** finding, run this protocol before acting:

### Step A — Simulate the reviewer's claim against the evidence

Pull the relevant context. Do not skip this — it is the difference
between a real critique and a posture:

1. Open the code the finding points at. Re-read it as if you didn't
   write it.
2. Re-read the relevant section of the **task spec** — especially the
   acceptance criterion the finding's `spec_anchor` references.
3. Re-read the relevant **prior `*.done.md`** files for decisions that
   may have already addressed (or constrained) what the reviewer is
   asking for.
4. Re-read the relevant section of the **TDD** for the design intent
   behind the spec.
5. Re-read `CLAUDE.md` for the convention the finding cites (or
   contradicts).
6. **Try to falsify the finding.** Can you construct a concrete
   scenario where the reviewer's claim does not hold? If yes, the
   finding may not be valid. If no, the finding likely is valid.

### Step B — Categorize the finding after simulation

After Step A, the finding falls into one of three buckets:

| After simulation             | What to do                                                                                                                                                            |
|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Holds — valid**            | Apply it (by severity rule below). No rebuttal needed.                                                                                                                |
| **Partially holds**          | Apply the part that holds. In `state.json.review_rounds[i].findings[j]`, record `addressed_with_modification: true` and a one-line note on what part you addressed vs. set aside. |
| **Does not hold — falsifiable** | Do **not** apply. Record a rebuttal — see Step C. Pass it to the next review round.                                                                                |

### Step C — When you rebut, the rebuttal must be evidence-based

A rebuttal is not "I disagree." A rebuttal is "I checked X, Y, Z, and
here's the concrete reason the finding does not hold." Format:

```
- Finding: <finding summary>
- Reviewer's claim: <reviewer's detail, condensed>
- Evidence I checked: <files / spec sections / prior done.md / TDD sections>
- Why the claim does not hold: <2-4 sentences with the concrete reason>
- What I am doing instead: <action, often "no change">
```

Record this in `state.json.review_rounds[i].rebuttals[]`. On the next
review round, include the rebuttals in the subagent's prompt context
so the reviewer can re-examine — *not* to argue, but to give them the
evidence you found. The next reviewer may still hold their ground; if
they do with the evidence in hand, the finding probably does hold,
and you re-apply your judgment.

### Step D — Apply, by severity, after challenge

For findings that **hold** after Step A:

| Severity   | What to do                                                                                                  |
|------------|-------------------------------------------------------------------------------------------------------------|
| `blocker`  | Apply. Non-negotiable. If you cannot apply it (e.g. you don't know how, or the fix conflicts with the spec), stop and surface to the user. |
| `major`    | Apply by default. If you genuinely cannot — because the fix would require deviating from the spec — surface to the user; do not silently defer. |
| `minor`    | Apply unless it conflicts with the spec. If you defer, log the deferral with a reason.                       |
| `nit`      | May defer. List all deferred nits in the handoff doc's "Deferred nits" section.                              |

### Step E — Constant disagreement → surface to the user

If after **two consecutive rounds** the same finding is being raised
and you have a defensible rebuttal that the reviewer is not accepting,
**stop the loop and surface to the user**:

> Round N: review still flags <finding>. My evidence-based rebuttal
> (see state.json round N-1) is <one-line summary>. The reviewer is
> not budging. This needs a judgment call. Options: (a) apply the
> finding as the reviewer asks, (b) accept my rebuttal and proceed,
> (c) revise the spec / TDD.

The same surface pattern applies if you hit any finding where you
**cannot honestly decide** whether it holds after Step A. Do not
guess. Surface and ask.

### After applying fixes — record, re-verify, then re-review

1. **Update the ledger.** Fill in the Resolution + Fix block for every
   finding in this round's `task-XX.review.md` section — the fix as a
   real diff hunk, or `n/a — rebutted` / `n/a — deferred`. The fix
   diffs you record here are what the next round's reviewer reads to
   scope its targeted re-review, so they must be accurate.
2. Re-run the `verify` stage end-to-end. Tests and lint must be green
   **before** the next review round. Do not invoke a new review with
   broken state.
3. Bump `attempts.review` in state.json.
4. Spawn a fresh subagent for round N+1 as a **TARGETED re-review**,
   passing: the review ledger, the explicit list of files/hunks you
   changed this round, the full current diff, this round's prior
   findings, and **any rebuttals from this round**. The reviewer
   verifies each prior finding is closed, deep-reviews your changed
   hunks, and regression-scans the rest (see "Scoping the review by
   round").

### Anti-patterns to avoid in disagreement

- **Arguing for the sake of arguing.** If you can't construct a
  falsification in Step A, the finding holds. Apply it.
- **Skipping Step A and rebutting on instinct.** Without evidence, a
  rebuttal is just opinion — exactly what fresh review exists to
  override.
- **Defer-via-rebuttal.** Calling a finding "invalid" because you
  don't want to fix it. That is dishonesty, not judgment.
- **Silent disagreement.** Applying a finding partially and pretending
  it was full compliance. Either fully apply, fully rebut, or surface.

---

## Round cap

**Hard cap: 3 review rounds.**

If round 3 returns `REQUEST_CHANGES` with any `blocker` or `major`
finding, stop. Do not invoke round 4. The likely root causes:

- The task spec is ambiguous or contradicts a project convention. Ask
  the user.
- The design itself doesn't work and the TDD needs revision. Ask the
  user to involve the tech-lead skill in its TDD-authoring or review
  capability for the design.
- The reviewer is anchoring on something the implementation can't
  satisfy. Show the user the latest findings and the latest diff; let
  them decide.

Surface the situation explicitly:

> Review round 3 still requests changes (blockers: <n>, majors: <n>).
> I'm stopping the auto-loop. Here are the unresolved findings: <list>.
> Likely root cause: <your best guess>. How would you like to proceed?

---

## Re-entering the loop from the pre-PR checkpoint

After the loop converges to APPROVE, the `checkpoint` stage may pause so
the user can inspect the ledger and ask for final changes before the PR
(see `references/workflow.md` → "Stage 6b — `checkpoint`"). Any change
the user requests during that pause is **new code that has not been
reviewed** — so it re-enters this loop rather than going straight to the
commit:

1. Implement the requested change.
2. Review it as a **targeted re-review** — it's a delta on
   already-reviewed work, scoped via the ledger exactly like a post-fix
   round.
3. Append the round to the ledger, apply / rebut findings, re-verify,
   and re-review until it converges to APPROVE again.
4. Return to the checkpoint and re-surface the updated ledger.

These are ordinary review rounds, recorded in `review_rounds` and the
ledger like any other. The 3-round cap applies **within a convergence
cycle**, not across the whole task: a fresh user request after an
APPROVE begins a new cycle with its own round count. The invariant is
simple — **nothing reaches the PR that didn't pass review**, and the
checkpoint is not an exception to it.

---

## When `review_skipped: true`

The user may explicitly tell you to skip review for a specific run
(e.g. "just implement and commit, skip the review loop"). When they do:

1. Set `review_skipped: true` in state.json.
2. Note in the PR description: "Implementation review was skipped at the
   user's request."
3. Note in the handoff doc's "Review findings & resolutions" section:
   "Review skipped at user request — no findings recorded."

Do not silently skip review even if "the change feels small." The
review is mandatory by default precisely because every implementer
thinks their own change feels small.

---

## What NOT to do in the review loop

- **Do not run the review yourself in your own context.** Spawn a fresh
  subagent every round.
- **Do not pass the subagent your own commentary on why the change is
  good** — that primes the reviewer.
- **Do not edit code while a review subagent is running.** Wait for the
  verdict, then act on it.
- **Do not re-use a subagent across rounds.** Fresh context each time.
- **Do not skip applying blocker / major findings** because you disagree.
  If you disagree, structure that disagreement as input to the next
  round.
- **Do not skip the ledger.** Every round gets a `task-XX.review.md`
  section with code evidence and fix diffs — it is the audit trail and
  the input that lets rounds ≥ 2 scope down. A round that fixes findings
  but doesn't record them forces the next round back to a full review.
- **Do not let a targeted re-review become a blank-slate review.** On
  rounds ≥ 2, pass the ledger + the changed-hunks list so the reviewer
  verifies the fixes and scans for regressions, rather than re-deriving
  every finding over unchanged code.
