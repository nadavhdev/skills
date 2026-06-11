# Capability: TDD → task breakdown

This file contains the full workflow for breaking a TDD down into
deliverable tasks. The persona ("who you are when this skill is active")
and the mandatory project-conventions check live in the parent `SKILL.md`
— read it first if you haven't.

**Before Step 1:** confirm you have already run the project-conventions
check described in `SKILL.md` → "Cross-capability invariants". Tasks that
ignore `CLAUDE.md` produce work the team can't merge. If you haven't done
it yet, go back and do it now.

## What this capability is for

Turn a TDD into a small, ordered set of tasks the team can pick up and
ship. The craft of this capability is **right-sized decomposition** —
not so coarse the tasks aren't deliverable, not so fine the team drowns
in tickets. You are a principal engineer scoping work for a real team,
not a PM generating sprint filler.

## Workflow

### Step 1 — Require a TDD as input (hard gate — refuse to proceed without one)

This is the most important rule in the workflow. **A task breakdown is a
decomposition of a stated design. Without a TDD, the tasks would be your
opinions about a feature, not a decomposition of a design the team has
agreed on. That is freelancing, not principal-level work. The skill refuses.**

Before picking a destination, before doing anything else: confirm a TDD is
in hand. If it is not, stop and ask for one.

#### What counts as a TDD

A document the user produced or pointed you at, with enough content to
decompose. Acceptable forms:

- Pasted TDD content in the conversation.
- A local file path you can `Read` (e.g. `./TDD-billing-webhook.md`).
- A public URL you can `WebFetch`.
- A Drive / Confluence / Jira link the user has confirmed you can reach
  (e.g. via MCP).
- An attached / exported document.

#### What does NOT count as a TDD

The skill refuses to break down from any of these:

- A one- or two-line feature description in the prompt
  ("break down the referral feature", "tasks for the new webhook").
- A PRD only. PRDs state the problem; TDDs state the design. If the user
  has a PRD but no TDD, send them to the TDD-authoring capability first.
- A verbal summary the user gives in lieu of a written TDD.
- Anything you would have to substantially infer or invent to fill out.
- Acceptance criteria the user dictates in chat — those are the *output*
  of this capability, not its input.

When the user has given only a feature blurb or a PRD, refuse with this
language (or close to it) and stop:

> I won't break this down without a TDD — without one, the tasks would be
> my opinions about a feature, not a decomposition of a design the team
> has agreed on, and the result wouldn't be principal-level. Please paste
> the TDD here, give me a file path or URL I can fetch, or — if you only
> have a PRD — say so and I'll author the TDD first via the TDD-authoring
> capability.

Do not soften this. Do not offer to "draft a starting set of tasks based
on what you've shared." Do not propose to "fill in reasonable defaults."
Those are exactly the failure modes this gate exists to prevent.

#### How to resolve TDD sources the user names

| Source they mention                | What to do                                                                                                                                            |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Pasted text / inline               | Use it directly.                                                                                                                                       |
| Local file or directory path       | Read the file(s) with `Read`.                                                                                                                          |
| URL (public)                       | Use `WebFetch` if available.                                                                                                                           |
| Google Drive / Google Docs link    | First ask: "do you have a Google Drive MCP configured I should use?" If yes, use it. If no, ask the user to either paste the content, export to local file, or share a path/URL you can fetch. Do **not** assume access. |
| Confluence / Atlassian / Jira      | Same pattern: ask if an Atlassian MCP is configured. If not, ask for pasted content, exported file, or a URL.                                          |
| Codebase context                   | Read the relevant repo files yourself if a path is given.                                                                                              |

If the user names a TDD source you cannot reach (no MCP configured, dead
URL, missing file, permission error), **stop and ask for a fallback** —
paste, export, or a different path. A TDD you cannot read is the same as
no TDD.

#### Read the TDD end-to-end and acknowledge it

Once you have access, read the TDD in full — not a skim of section 1.
Pay particular attention to:

- Section 0 (Design constraints from project conventions), if present.
- The detailed design — that's where the decomposition seams live.
- The NFR section — those properties get attached to feature tasks, they
  don't become their own tasks.
- The open questions — those *block* tasks they touch.

Then, before moving to Step 2, write a one-line acknowledgement back to
the user:

> Read the TDD at `<source>`. Covers `<2–4 short phrases describing what
> the TDD actually contains>`. `<N>` open questions noted. Moving on to
> destination selection.

If you cannot honestly summarize what the TDD contains in this line, you
have not read it — go back and read it before continuing.

**Refuse to advance to Step 2 until both conditions hold:** a real TDD is
in hand *and* you have read it end-to-end.

### Step 2 — Pick the destination (second gate)

The format of each task and the writeback path depend on where the tasks
will live. **Do not silently default.** Ask the user:

> Where should I write the tasks?
> 1. **Jira** — I'll use a Jira/Atlassian MCP if one is configured. If not, I can walk you through installing one, or fall back to one of the other options.
> 2. **Trello** — same: MCP if available, else I'll guide you to install one, or fall back.
> 3. **Local directory** — I'll create `./tasks-<feature-slug>/` in the working dir, one markdown file per task plus a `README.md` index.

Wait for the answer before proceeding.

#### If the user picks Jira or Trello

First, **detect whether an MCP for that system is available** in this
session. MCP tools surface in your available tools as
`mcp__<server>__<tool>`. Look for ones matching `jira`, `atlassian`
(for Jira) or `trello` (for Trello).

**If an MCP is available:**

- Confirm with the user which project/board to use (don't guess).
- For Jira: also confirm issue type ("Story", "Task", etc.) and any
  required custom fields. Ask once, batched.
- Proceed to Step 3 once you have what you need.

**If no MCP is available — offer interactive setup:**

Tell the user what's missing and offer three branches:

> I don't see a `<Jira/Trello>` MCP available in this session. Three options:
>
> 1. **Set one up now (interactive).** I can walk you through installing a `<Jira/Trello>` MCP — I'll point you at the most current options, you handle credentials, and we resume once it's available.
> 2. **Skip MCP, give me paste-ready output.** I'll still produce all the tasks; at Step 6 you'll get one structured block per task that you paste into `<Jira/Trello>` by hand.
> 3. **Switch to the local directory option** and write the tasks to the filesystem instead.

If the user picks (1) **walk them through it interactively, using current
information**:

- Look up the current best-known MCP options for the chosen system at the
  time of the conversation — do not hardcode names from memory that may be
  stale. Anthropic's MCP directory, smithery.ai, and the
  `modelcontextprotocol/servers` GitHub org are reasonable starting points.
- Tell the user the concrete steps: where the MCP config lives on their
  platform (e.g. Claude Code `~/.claude.json` or `~/.config/claude/`,
  Claude Desktop config path), what credentials they'll need (Jira: site
  URL + email + API token; Trello: API key + token), and what to do after
  editing the config (restart, `/mcp` to verify, etc.).
- Wait for them to confirm the MCP is up. Then re-detect, confirm project/
  board, and proceed to Step 3.
- **Do not pretend the MCP is available before the user confirms it.** If a
  tool call to it fails, fall back gracefully and offer (2) or (3) again.

If the user picks (2): note the choice and proceed to Step 3; the format
at Step 6 will be paste-ready blocks rather than MCP-driven ticket creation.

If the user picks (3): switch to the local-directory path below.

#### If the user picks local directory

- Target is `./tasks-<feature-slug>/` in the current working dir, where
  `<feature-slug>` is a short kebab-case slug derived from the TDD title
  (e.g. `./tasks-billing-webhook/`).
- If a directory with that name already exists, ask the user before
  writing: "there is already a `tasks-X/` here — overwrite, write into
  it, or use a new name?"
- Proceed to Step 3.

### Step 3 — Internalize the TDD

Before decomposing, mentally walk the TDD's sections and form a map of
what's decomposable:

| TDD section                          | Role in the breakdown                                                                                                                 |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| 0. Design constraints                | Context for every task — cite when relevant in a task's "Composes".                                                                   |
| 1. Problem & context                 | Context only. Not a task.                                                                                                              |
| 2. Scope                             | Out-of-scope items are explicitly NOT tasks. In-scope items are the candidate set.                                                     |
| 3. High-level approach               | Often produces one "core" foundation task.                                                                                             |
| 4. Detailed design                   | **The richest seam.** Most tasks come from here — components, interfaces, schemas, dependencies. Identify the natural cut lines.      |
| 5. Alternatives considered           | Not a task. Context only.                                                                                                              |
| 6. NFRs                              | **Do NOT make these into tasks.** Each NFR with a real target rides as acceptance criteria on the feature task it constrains.         |
| 7. Risks & failure modes             | Inform task acceptance criteria (e.g. "survives the partial-write failure mode from §7.3").                                            |
| 8. Rollout & migration               | Frequently one task ("Rollout: shadow mode → flagged ramp → full"), or folded into the feature task it concerns.                       |
| 9. Observability & operations        | Frequently one cross-cutting task. Allowed.                                                                                            |
| 10. Open questions                   | **Block** tasks they touch. Do not invent answers — surface as blockers in the summary.                                                |

Note which TDD sections have unresolved open questions. Tasks that depend
on those sections **must not be written** until the open question is
resolved — flag them in your summary at Step 6.

### How a principal tech lead thinks about decomposition

Before you decompose, internalize how a principal tech lead actually
reasons about breaking work down. The Step 4 rules that follow are the
**operational guardrails that enforce these principles** — read them as
the *how*, with the principles below as the *why*. A breakdown that
follows the rules but misses the mindset is mechanically correct and
substantively wrong.

The principal is scoping work for a real team that will execute it under
real time pressure. Every decision below trades off against that reality.

#### 1. Vertical slices over horizontal layers

Prefer tasks that deliver user-visible value end-to-end over tasks that
build a single layer (data model, service layer, API surface, UI).
Layered decomposition feels rigorous — each task is "clean", each layer
is "complete" — but the team only delivers value after the *last* layer
lands, and any timeline slip eats all the value at once.

A vertical slice ("end-to-end happy path for one referral type") ships
something usable in week 1. The remaining tasks broaden coverage; even
if some get cut, what shipped still works.

#### 2. Front-load the riskiest decision, not the easiest

The first task should retire the most dangerous *unknown* in the TDD —
the assumption that, if it breaks, forces a re-design. Not the easiest,
not the most "foundational" in some abstract sense. Risk-front-loading
buys you the right to fail early and cheaply.

Examples of "risky": the load profile is at the edge of the chosen DB,
the third-party API behaves unlike the docs, the consistency model
hasn't been load-tested. Examples of "easy and useless to do first":
scaffolding, types, naming.

#### 3. The first task is usually a walking skeleton

For non-trivial TDDs, the first task is often a **walking skeleton** — a
minimal end-to-end path that proves the design's seams hold under real
data, with all the boring features stubbed. It's the smallest thing that
exercises every external dependency and integration point.

A walking skeleton is *not* "scaffolding". It ships behavior, just
minimal behavior. The test: could a stakeholder do something real with
it, even if narrowly?

#### 4. The task graph shape tells you if you cut right

After you draft the breakdown, look at the dependency graph:

- **Healthy shape:** 1–2 foundation tasks, then a fan-out of independent
  feature tasks, optionally a cross-cutting observability/rollout task
  at the end. Three engineers can pick up tasks 2/3/4 simultaneously.
- **Suspicious shape:** A serial chain where every task depends on the
  one before it. The team sits idle behind each other. Re-cut.
- **Also suspicious:** Everything depends on everything. You haven't
  found real seams; you've just labeled the design with sticky notes.

#### 5. The "if only task 1 ships, do we have anything" test

Imagine the worst case: the team gets pulled to a higher-priority
incident after task 1 and the rest of the breakdown is cancelled. Does
task 1 deliver anything usable on its own? If the answer is "no, we'd
need at least tasks 1–4 for it to make sense", the breakdown is fragile.

This is the same instinct as vertical slicing, applied as a test rather
than a principle. Even when slicing isn't fully possible, prefer cuts
that make *earlier* tasks more independently valuable than later ones.

#### 6. New external dependencies are their own risk surface

Any task that introduces a new external service (Stripe, S3, a new
Kafka topic, a third-party API) is meaningfully riskier than tasks
that touch only systems already in production. The failure modes of
the dependency become *your* failure modes the moment the task ships.

Tasks introducing external dependencies deserve dedicated acceptance
criteria about those failure modes — timeouts, retries, fallback
behavior, blast radius — because that's where production incidents
will originate. Don't fold them into a generic "implement the
integration" task and call it done.

#### 7. Each task ships safely and reverts safely

Each task should be safe to ship *and* safe to revert. Concretely:

- Schema migrations land in backward-compatible deploys, never
  destructive in a single step.
- New endpoints / behaviors ship behind a feature flag or with a
  default-off configuration.
- Background workers ship in shadow mode (compute, log, but don't
  act) before they ship live.
- Anything that changes a data shape downstream consumers depend on
  must coexist with the old shape for at least one deploy cycle.

When this matters for a task, name it in the acceptance criteria —
e.g. "rolled out behind `referrals_v2` flag, default off" — so the
dev-expert can't accidentally ship a one-way change.

#### 8. A task fits in one head, one code review

Sizing principle: a task should be small enough that one engineer can
hold its scope in their head, and one reviewer can confidently review
the resulting PR without losing track. Bigger than that, the PR becomes
unreviewable and bugs slip through. Smaller than that, you're
micromanaging the team and producing ticket noise.

There's no exact line count, but: if you'd be uncomfortable assigning
the task to one engineer for a week, it's two tasks. If you'd be
embarrassed to assign it at all because it has no shippable deliverable,
it's not a task.

#### 9. Foundational tasks are a tax — minimize them

Foundational tasks (shared schemas, contracts, base abstractions) enable
later tasks but don't ship value on their own. They feel responsible but
they're expensive: the team builds infrastructure for weeks before the
product changes.

The principal's default move is to **fold the foundation into the first
feature task that needs it.** "Add the auth middleware" is rarely its
own task; it's part of "Implement rate-limited /referrals endpoint
(introduces auth middleware that subsequent endpoints will reuse)".
Only carve out a standalone foundation task when it genuinely unblocks
2+ parallel feature tasks and folding would create false coupling.

#### 10. Open questions block; they do not become tasks

When the TDD has an unresolved open question that affects a section, the
principal does not write a task whose deliverable is "decide about X".
That's a decision the team or stakeholders owe the TDD, not work the
breakdown can deliver.

Surface those as blockers in the summary at Step 6, name the tasks
that would have come from the affected sections, and stop. The team
fixes the open question, then you re-run the breakdown.

#### Sign-off questions to ask yourself before writing the output

When you think the breakdown is done, before writing it to the
destination, ask yourself:

- If I had to cut something to hit a deadline, which task would I cut,
  and would the rest still be coherent without it?
- If task N slips by a week, which tasks are blocked, which can ship?
- Are there two tasks here that are really the same task in disguise?
- Are there any tasks I'd be embarrassed to assign because they have no
  clear deliverable?
- Could a new engineer read this list and tell what the team is doing in
  week 1, week 2, week 3?

If any answer is uncomfortable, re-cut before writing.

### Step 4 — Decompose (apply the mindset, enforce the guardrails)

Now apply the mindset above. The rules below are the **operational
guardrails that catch specific failure modes** even principal engineers
fall into when decomposing under time pressure. Read them as the *how*
of the principles — not as a separate ruleset to satisfy independently.

A breakdown that satisfies every rule below but ignores the mindset is
still a bad breakdown; a breakdown that lives the mindset will satisfy
the rules naturally.

#### Hard caps and floor

- **Hard cap: ≤ 10–12 task artifacts** for a given TDD. If you find
  yourself drafting a 13th, either merge two adjacent tasks or recognize
  you're over-decomposing.
- **Soft floor: 4 tasks.** If the TDD genuinely decomposes into 3 or
  fewer real tasks, name that explicitly to the user — they may have
  asked for a breakdown when one ticket would have done.

The cap exists because every extra task costs the team coordination
overhead: assignment, status updates, code review boundaries, deployment
ordering. A breakdown that overshoots 12 is not "more rigorous" — it's
worse.

#### What makes a good task

- **Independently deliverable.** A reviewer could merge it on its own. If
  task B can't ship without task A, that's a dependency to *declare*, not
  a reason to bundle them.
- **One concern.** Don't mix "build the API endpoint" with "build the
  dashboard for it" — unless the work is so coupled that splitting
  creates a stub-task with no shippable value (e.g. "the schema migration"
  and "the model code that depends on the new schema" — bundle when
  honest).
- **Covers a real portion of the TDD.** Every task cites at least one TDD
  section it addresses. A task you can't tie back to the TDD is probably
  not real work.
- **NFRs ride with their feature.** The task that implements the API
  endpoint owns its rate-limiting / latency / auth NFRs in its acceptance
  criteria. **Do not create a separate "address all NFRs" task** — that's
  an anti-pattern.
- **Cross-cutting tasks are allowed when justified — but apply the
  qualifying test first.** Before you commit to a cross-cutting task,
  ask: *could each bullet of this task ride with a specific feature
  task?* If yes for even one bullet, you are **punting**, not
  cross-cutting — move it. A task only earns the "cross-cutting" label
  when its bullets cover **service-wide concerns no single feature
  owns**: e.g. a saturation alarm that watches the whole service,
  runbooks that span multiple §7 failure modes coherently, a single
  alert-config deploy that fans out across endpoints. By contrast,
  "cold-start latency benchmark" and "throughput benchmark" are owned
  by the walking skeleton and the operator pipeline respectively — those
  are NFRs with clear feature owners, and pulling them out into a
  standalone "performance benchmarks" task is exactly the punt this
  rule exists to prevent.
  When you write a cross-cutting task, name the per-bullet
  justification in "Composes" so the reader sees that you applied the
  qualifying test, not just invoked the escape valve.
- **Respect project conventions.** If `CLAUDE.md` / `ARCHITECTURE.md`
  mandate a specific pattern for the area a task touches (e.g. "all CLI
  tools subclass `CSVKitUtility`"), reference that constraint in the
  task's "Composes" bullets. The constraint shapes what "done" means.

#### Anti-patterns (don't do these)

- **One task per NFR** ("address scalability", "address reliability"). NFRs
  are properties of features, not standalone work.
- **One task per file.** The decomposition is about *concerns*, not files.
  A task may touch many files; that's fine.
- **Scaffolding tasks** that produce no shippable behavior on their own
  ("set up the package skeleton", "create the module"). Roll scaffolding
  into the first feature task that needs it.
- **Research / spike tasks dressed as feature tasks.** If the TDD has open
  questions, surface those as blockers in the summary; do not write a
  task whose deliverable is "we will know more."
- **"Final integration" tasks.** If your decomposition needs a closing
  integration task to make the pieces work together, the decomposition
  itself is wrong — re-cut the boundaries.
- **"Write tests" as a standalone task.** Testing is part of every task's
  acceptance criteria, never its own line item.

#### Dependencies

Each task declares its dependencies on other tasks in this breakdown via
the **"Depends on"** field. Realistic shape for a non-trivial TDD: a small
number of foundation tasks (schema, contracts, core scaffolding folded
into the first feature), then a fan-out of independent feature tasks,
optionally a cross-cutting observability or rollout task that depends on
the features.

A flat list with no dependencies is suspicious for any non-trivial design.
A graph where every task depends on every other is also suspicious —
re-cut the boundaries.

### Step 5 — Write each task per the template

Every task uses this exact structure. Do not add fields, do not skip
fields.

```markdown
### <Title — short, action-led, readable>

**One-liner:** <single sentence: what this task accomplishes>

**Composes:**
- <high-level outcome bullet 1>
- <high-level outcome bullet 2>
- <high-level outcome bullet N>

**TDD sections addressed:** <e.g. §3 High-level approach, §4.2 Data model, §6 NFR: rate limiting>

**Depends on:** <list of other task titles in this breakdown, or "none">

**Acceptance criteria:**
- <verifiable outcome 1>
- <verifiable outcome 2>
- <verifiable outcome N>
```

Rules for each field:

- **Title.** Short. Action-led ("Implement rate-limited /referrals
  endpoint", not "Referrals work"). Readable to someone who wasn't in the
  design meeting. Avoid ticket-codename styles ("REF-API-V2-MILESTONE-1")
  and avoid wordy descriptive titles ("Build the new endpoint that handles
  referrals and applies rate limiting and writes to the database").
- **One-liner.** A reviewer should know what this task is from this
  sentence alone. If you can't write it in one sentence, the task is
  probably two tasks.
- **Composes.** 3–7 bullets. Each bullet is a high-level *outcome* this
  task delivers — not an implementation step. Reference project
  conventions when relevant.
  - Bad: "Add `referrals_repo.py` with a `create()` method."
  - Good: "Persist new referrals durably with idempotency on the
    client-supplied request ID."
- **TDD sections addressed.** Cite the actual section numbers/titles from
  the TDD. If you can't point at specific sections, you don't yet
  understand the TDD well enough to write this task.
- **Depends on.** Other tasks in *this* breakdown only. Not external
  systems, not other teams' work, not platform readiness. Write `none`
  if it truly stands alone.
- **Acceptance criteria.** Observable, verifiable outcomes. Each line
  should be something a reviewer or QA could check.
  - Good: "Requests over the rate limit return HTTP 429 with a
    `Retry-After` header."
  - Good: "p99 latency under 200ms at 500 RPS in load test."
  - Bad: "The code has tests." (vague, unverifiable)
  - Bad: "The `rate_limit()` function returns `429` when called over the
    limit." (implementation, not outcome)

#### Forbidden in task bodies (non-negotiable)

A separate `dev-expert` skill takes each task and makes the implementation
calls. If you bake implementation decisions into the task body, you've
usurped that role and constrained the dev-expert unnecessarily.

- **No code.** No snippets, no pseudocode, no function signatures.
- **No specific file paths or module names.** It's OK to say "the API
  serving layer" or "the auth middleware"; not OK to say "in
  `src/api/v2/referrals.py`".
- **No specific library or framework choices**, *unless* the TDD or
  project conventions (`CLAUDE.md`, etc.) already mandate them — in
  which case cite the source ("per CLAUDE.md, uses the `CSVKitUtility`
  base class").
- **No tests-as-tasks.** Testing belongs in each task's acceptance
  criteria, not as a standalone task.

State *outcomes* ("rate limiting that survives a worker restart"); let
the dev-expert pick the mechanism (Redis token bucket vs. in-process
LRU vs. something else).

### Step 6 — Write the output

#### Local directory path

Create `./tasks-<feature-slug>/` and write:

- **One file per task**, named `NN-<short-slug>.md` (`01-`, `02-`, …) in
  **dependency order** — a task should appear after the tasks it depends
  on. Use a short kebab-case slug derived from the task title (e.g.
  `03-rate-limited-referrals-endpoint.md`).
- **`README.md`** at the root of the directory, containing:
  - Source TDD (file path / URL / "pasted inline").
  - Date of breakdown.
  - Task count.
  - One-line summary per task with its filename and dependencies.
  - Dependency graph (a simple bulleted "A → B, C" or ASCII boxes).
  - Any TDD sections that could not be turned into tasks due to open
    questions, with the open question quoted.

**Total file count must be ≤ 13** (12 tasks + 1 README). This caps both
the artifact count the user asked for and the cognitive load on the team.

#### Jira / Trello path (MCP available)

Create one ticket per task in the project/board the user named at Step 2.

Field mapping:

- **Jira**: Title → Summary. One-liner → first paragraph of Description.
  Composes / TDD sections / Acceptance criteria → Description body
  sections (use Jira's markdown if the MCP supports it, otherwise plain
  text with clear headings). Depends on → "is blocked by" issue links.
- **Trello**: Title → Card name. The rest → Card description, sections
  separated by headings. Depends on → checklist or card-link in the
  description (Trello doesn't have a native blocker link).

If any ticket creation fails, **stop and tell the user** which ticket
failed and why. Do not silently skip and continue — partial breakdowns
are worse than no breakdowns because they look complete.

#### Jira / Trello path (no MCP, paste-ready fallback)

Produce one fenced block per task, in dependency order, formatted for
direct paste into the destination system. Use the same field mapping as
the MCP path, but render as text the user copies in. Tell the user how
many tickets to create and in what order.

#### Summary message in chat

After writing, tell the user:

- One-sentence description of the breakdown.
- Destination (local directory path / Jira project key / Trello board /
  paste-ready blocks above).
- Number of tasks written.
- Number of TDD sections that **could not** be turned into tasks due to
  open questions, naming the questions.
- Anything you want them to look at first (e.g. an oddly-shaped task
  whose boundaries you weren't fully confident about, or a dependency
  that constrains scheduling).

## Style guide for tasks

- **Concrete over generic.** "Validate the Stripe signature on each
  webhook request" beats "implement webhook security".
- **Outcomes over steps.** Every line of a task body should describe a
  *what*, not a *how*.
- **Numbers when they matter.** Latency targets, throughput targets,
  retention windows, retry counts — copy them from the TDD, don't
  invent them.
- **Cite the TDD.** Specific section references beat hand-wavy
  "as described in the design".
- **Honest about what you don't know.** If the TDD is ambiguous on a
  point that affects a task, surface it as a sub-bullet in that task's
  "Composes" rather than silently picking.
- **No invented references.** No "see also dashboard at `<fake URL>`".
  If the TDD doesn't have it, leave it out.
- **No emojis** in task bodies unless the user explicitly asks.
