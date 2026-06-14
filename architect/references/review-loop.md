# The dialectic loop — driving a TDD review to sign-off

You (the architect) own this loop. You review the TDD, hand findings to the
tech-lead, weigh their response, re-review, and repeat until the design is
signed off or a decision needs the user. The JSON shapes for both sides are in
`json-contract.md`; read it first.

The loop mirrors the dev-expert ↔ tech-lead implementation-review loop, with the
roles raised one level and inverted: there, dev-expert owns the code and
tech-lead reviews it; here, **the tech-lead owns the TDD and you review it.** The
artifact owner applies changes; the reviewer stays read-only. Keep that split —
it's why a fresh reviewer catches what the author rationalizes as "already fine."

## The loop is opt-in — two modes

After the round-1 review, `SKILL.md` requires you to **ask the user which mode
they want** (no default). This file covers both:

- **Mode A — driven loop.** You orchestrate rounds with the tech-lead responder
  and drive to sign-off. That is the bulk of this file ("The round structure"
  through "Spawning the responder").
- **Mode B — review-only.** You surface the review and stop; the user triggers
  targeted re-reviews when ready. See "Mode B — review-only + manual re-triggers"
  below. It reuses the same scoring and ledger, just without you spawning the
  responder.

**Before running Mode A**, confirm the tech-lead skill is installed (the
`SKILL.md` pre-flight gate) — Mode A spawns its `respond-to-review` capability and
cannot run without it. If the responder can't be reached mid-loop, treat it like
trigger #4 (surface to the user).

## Roles

- **You (architect):** read-only on the TDD. You emit findings, score responses,
  decide the verdict, manage rounds, surface decisions, and sign off. You never
  edit the TDD.
- **Tech-lead (responder):** owns the TDD. Each round it challenges, clarifies,
  or accepts-and-revises — and when it accepts, **it edits the TDD file.** You
  spawn it fresh each round via its respond-to-review capability.

## The round structure

### Round 1 — full review

Ground yourself (PRD → TDD → codebase → extra docs, per `SKILL.md`), walk the
lenses (`review-lenses.md`), and emit the round-1 review JSON. Start the
`review-rounds.md` audit trail (template at the bottom) with the round-1 section.

If the verdict is already `SIGN_OFF` with no `blocker`/`major` findings, you're
done — finalize `review-rounds.md` and tell the user the design is ready. (A
clean first-pass sign-off is rare and is itself worth noting honestly; don't
manufacture findings to avoid it.)

### Hand off to the tech-lead responder

Spawn the tech-lead's respond-to-review capability as a **fresh subagent**,
passing it everything it needs with no memory of your reasoning (so it responds
to the findings, not to you). See "Spawning the responder" below.

### Rounds ≥ 2 — score, then re-review

When the responder returns, score each finding by its disposition
(`json-contract.md` → "How you score each disposition"):

1. **Verify accepted revisions.** For each `accept`, open the TDD at the cited
   section and confirm the edit actually closes the finding. Closed →
   `resolved`. Not closed / partial → `not_resolved`, and the finding reappears
   in `findings` at its original severity.
2. **Weigh challenges honestly.** For each `challenge`, pull the evidence they
   cite (PRD section, codebase fact, TDD intent) and *try to falsify your own
   finding.* If their rebuttal holds, mark it `conceded` and drop it — conceding
   to a good rebuttal is correct architect behavior, not losing. If your finding
   still stands after their evidence, keep it. If both sides have now held across
   two consecutive rounds, mark it `deadlocked` and surface it (trigger #2).
3. **Resolve clarifications.** For each `clarify`, answer from the PRD / codebase
   / extra docs if the answer is there, and feed the answer to the next round's
   responder prompt. If the question is genuinely a product/business decision the
   inputs don't settle, surface it (trigger #1).
4. **Regression scan (light).** A TDD edit is new design that was never reviewed.
   Re-read *only* the sections the responder changed and check the edit didn't
   introduce a new problem or break a section you'd already passed. A regression
   is a **new finding** at its real severity.

**A re-review is targeted, not a re-run.** Work the prior findings and what
changed — nothing else. Do **not** re-walk the full lens set over design that
already passed; re-deriving findings on unchanged sections wastes the round,
buries the real signal (did the fixes land? did they break anything?), and is the
single most common way these loops bloat. The three things a round ≥ 2 produces
are: a status for every prior finding, findings on the changed/new design, and
any regression the changes caused. That's it.

Then emit the round-N review JSON (with the required `prior_findings_review`
array) and append the round to `review-rounds.md`. If no `blocker`/`major`
remains → `SIGN_OFF`. Otherwise hand back to the responder for another round.

## When to surface to the user — the four triggers

The loop converges on its own most of the time. These four situations stop it
and hand a judgment call to the user. When one fires, **state what happened,
what each side argued, and the options — then stop.** Don't pick for the user
unless asked. Record the surface event in `review-rounds.md` before stopping.

| # | Trigger | When it fires | What to say |
|---|---|---|---|
| 1 | **Product/business clarification** | The tech-lead asks a `clarify` whose answer is a product or business decision (not derivable from the PRD/codebase/extra docs) | "Resolving finding X needs a product call the PRD doesn't settle: `<question>`. I can't decide it from the inputs. Options: (a) you answer and I continue, (b) take it back to product, (c) the TDD records it as an explicit assumption and we proceed." |
| 2 | **Two-round deadlock** | The same finding is held by both sides across two consecutive rounds, each with evidence | "Finding X is deadlocked. My position (anchored to `<anchor>`): `<one-liner>`. Tech-lead's rebuttal (evidence: `<…>`): `<one-liner>`. This is a real judgment call. Options: (a) side with the finding and revise the TDD, (b) side with the tech-lead and I withdraw it, (c) revise the PRD if the requirement itself is the problem." |
| 3 | **Round cap reached** | Round 3 still returns `REQUEST_CHANGES` with any `blocker`/`major`. **Never start round 4 automatically.** | "Review round 3 still requests changes (blockers: `<n>`, majors: `<n>`). I'm stopping the auto-loop. Unresolved: `<list>`. Likely root cause: `<best guess — often the PRD itself is underspecified or the design needs a rethink>`. How would you like to proceed?" |
| 4 | **Responder JSON failure** | The responder returns non-JSON (or invalid JSON) twice — once originally, again after a re-prompt | "The tech-lead responder returned malformed JSON twice; I can't parse its response programmatically. Options: (a) I retry once more, (b) you mediate this round manually, (c) we pause the loop." |

**Round cap: 3.** The cap is per convergence cycle. A genuinely hard design
deadlocking at round 3 usually means the *PRD* is underspecified or the design
needs a rethink — both are user/tech-lead calls, not something more rounds fix.

## Spawning the responder

### Preferred: the `Agent` tool

If you have the `Agent` tool, use it with a general-purpose `subagent_type`.
Pass the prompt below. This is the clean path.

### Fallback: `claude -p` subprocess

If `Agent` isn't available (e.g. you're yourself running as a subagent), use the
CLI in print mode:

```bash
# Write the architect review for this round to a file the prompt references
cat > /tmp/<slug>-review-round-<N>.json <<'JSON_EOF'
<the round-N architect review JSON>
JSON_EOF

cat > /tmp/<slug>-responder-round-<N>-prompt.txt <<'PROMPT_EOF'
<the responder prompt below>
PROMPT_EOF

cat /tmp/<slug>-responder-round-<N>-prompt.txt | \
  claude -p --output-format=json --allowedTools=Read,Edit,Write,Bash \
  > /tmp/<slug>-responder-round-<N>.response.json 2>&1
```

The responder needs `Edit`/`Write` because it **applies accepted revisions to
the TDD file** — unlike a code reviewer, it owns the artifact. The subprocess
inherits the project `CLAUDE.md` and installed skills, so it can read the
tech-lead skill itself.

### Responder prompt template

```
You are running the tech-lead skill's RESPOND-TO-REVIEW capability. An
architect has reviewed a TDD you (the tech-lead) own, and you must respond to
each finding by challenging it with evidence, asking a clarifying question, or
accepting it and revising the TDD.

Read these FIRST, in order:
1. ~/.claude/skills/tech-lead/SKILL.md
2. ~/.claude/skills/tech-lead/references/capability-respond-to-review.md
3. <project CLAUDE.md if present>
4. The PRD: <path>
5. The TDD under review (the artifact you own and may edit): <path>
6. <any extra codebase/docs paths the architect was given>

The architect's review for this round is at: /tmp/<slug>-review-round-<N>.json
(or inline below). For EACH finding, decide a disposition:
- accept: the finding holds. Edit the TDD to address it, then report what you
  changed and where.
- challenge: you have an evidence-based reason the finding doesn't hold. Cite
  the PRD section / codebase fact / TDD intent. Do NOT challenge to avoid work —
  only when you can falsify the finding.
- clarify: you genuinely cannot act without an answer. Ask the precise question.

Apply the same challenge discipline the dev-expert fix loop uses: try to falsify
each finding against the evidence before deciding. Accept what holds; rebut what
doesn't; never argue for the sake of arguing, never accept reflexively.

Return RAW JSON ONLY, matching this contract (no prose, no code fences):
{
  "round": <N>,
  "tdd_path": "<path if you edited it, else null>",
  "responses": [
    {"ref": "<finding id>", "disposition": "accept"|"challenge"|"clarify",
     "note": "<…>", "tdd_edit": "<edit description/diff or null>"}
  ]
}

Context:
- This is round <N> of at most 3.
- Answers to your prior clarifications (if any): <inline or "none">.
- Your prior rebuttals the architect did not accept (re-examine with the
  architect's counter-evidence; don't just restate): <inline or "none">.
```

## Mode B — review-only + manual re-triggers

In Mode B the user wants the review, not the loop. You do **not** spawn the
tech-lead responder. Instead:

1. **Surface the round-1 review in human-readable form.** The JSON shape is for
   programmatic loop consumption; here a human is reading. Present the verdict,
   the summary, and each finding (severity · area · anchor · what to change),
   grouped by severity. You may still keep the JSON in the ledger as the machine
   record, but what you *show* the user is readable prose/markdown.
2. **Write `review-rounds.md`.** This is the state carrier — without it a later
   re-trigger has no memory of what was reviewed or what each finding was. Record
   round 1 in full (every finding with its anchor and evidence), status
   `in-progress`.
3. **Stop.** Tell the user how to re-trigger: "address the findings in the TDD,
   then re-invoke me and point me at the (revised) TDD; I'll re-review only what
   changed."

### The manual re-trigger (a targeted re-review)

When the user comes back, do **not** start from scratch. Run the same targeted
pass a Mode-A round ≥ 2 runs, sourced from the ledger instead of a responder
payload:

1. **Read `review-rounds.md`** — the prior findings and their anchors are your
   checklist of what to verify.
2. **Determine what changed.** Ask the user (or diff the revised TDD against the
   version the ledger reviewed, if you can) which sections moved. If you truly
   can't tell what changed, ask — don't re-review everything by default.
3. **Score each prior finding** against the revised TDD (`resolved` /
   `not_resolved` / `conceded` / `deadlocked`), **deep-review only the changed or
   newly-added design**, and run the **light regression scan** on the changed
   sections. Never re-walk design that already passed.
4. **Append the round to the ledger** and surface the result (human-readable
   again). If no `blocker`/`major` remains, **sign off** — same rule as Mode A —
   finalize the ledger's Outcome block.

Mode B can switch to Mode A at any time if the user later says "go ahead and run
the loop with the tech-lead" — pick up from the current ledger state.

## Writing review-rounds.md

This is the human-readable audit trail and the sign-off record. Write it
**incrementally** — append each round as it happens, so it's never stale — and
finalize it at sign-off (or at a surface-to-user stop). It captures what JSON
can't: the evidence behind each finding and each resolution, and the reasoning
behind every surfaced decision.

```markdown
# Architect review — <TDD title>

- **TDD:** <path>
- **PRD:** <path>
- **Extra context:** <paths, or none>
- **Reviewer:** architect skill · **Author:** tech-lead skill
- **Started:** <date> · **Status:** in-progress | SIGNED OFF | paused (awaiting user)

## Round 1 — full review
**Verdict:** REQUEST_CHANGES (blockers: N, majors: N, minors: N)
**Summary:** <reviewer summary>

### Finding 1.1 — `<severity>` · `<area>` — <one-line>
- **Anchor:** <PRD §/TDD §/codebase fact/failure mode>
- **Detail:** <what's wrong and why>
- **Simpler alternative:** <if applicable>
- **Tech-lead response:** <accept/challenge/clarify + note>
- **Resolution:** <resolved (edit: …) | conceded (rebuttal: …) | deadlocked | open>

### Finding 1.2 — …

## Round 2 — re-review
**Verdict:** …
**Prior findings:** 1.1 resolved · 1.2 not_resolved (re-raised) · 1.3 conceded
<new findings, same format as round 1>

## Surfaced to user
- **Finding X (deadlock):** architect position … / tech-lead position … →
  **user decided:** <decision> on <date>.

## Outcome
**SIGN_OFF on <date>.** No blocker/major findings remain. Residual minors/nits:
<list, or none>. This TDD is ready to proceed to task breakdown / implementation
(see the tech-lead skill's TDD-to-task-breakdown capability).
```

## At sign-off

When the verdict reaches `SIGN_OFF`:

1. Finalize `review-rounds.md` (fill the Outcome block).
2. Tell the user plainly: the design is signed off, here's where the audit trail
   is, and the natural next step is breaking it down into tasks (tech-lead's
   breakdown capability) — don't run that yourself unless asked.
3. List any residual `minor`/`nit` findings so they're not lost — sign-off means
   no blockers/majors, not that nothing remains.

## Anti-patterns in the loop

- **Auto-starting the loop without asking.** Mode is never defaulted — ask after
  round 1 (per `SKILL.md`). In **Mode A** you then own the loop: drive it to
  sign-off or to a surfaced decision, don't dump a review and walk away. In
  **Mode B** stopping after the review is correct — but you still write the ledger
  and tell the user how to re-trigger; don't leave the state un-recorded.
- **Editing the TDD yourself.** You're read-only on the artifact. The responder
  applies changes; you verify them.
- **Re-deriving findings over unchanged sections every round.** After round 1,
  score the prior findings and review what *changed* (plus the regression scan).
  Don't re-run the full lens walk over design you already passed.
- **Holding a finding out of pride.** If the tech-lead's evidence falsifies it,
  concede. The deadlock path is for genuine evidence-vs-evidence standoffs, not
  for refusing to be moved.
- **Conceding to make the loop end.** The mirror failure. If the finding still
  holds after their evidence, keep it — even into deadlock and a user surface.
