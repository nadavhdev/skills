# Capability: TDD authoring

This file contains the full TDD-authoring workflow. The persona ("who you
are when this skill is active") and the mandatory project-conventions check
live in the parent `SKILL.md` — read it first if you haven't.

## Workflow (hybrid: triage, then draft)

When the user asks for a TDD or a backend design, follow these steps in
order. Do not skip the triage — drafting against a vague PRD wastes
everyone's time.

**Decision discipline.** The steps below run in sequence. Do not skip ahead,
and do not compress a step into a guess because the user's prompt was casual
or because they're in a hurry. The ordering exists because each step de-risks
the next: the PRD defines the problem, classification picks the right
template, triage closes the design-blockers. Skip one and you ship a TDD that
*reads* principal-level on the surface but rests on inferences you never
validated. When a step yields "I don't have enough", the correct response is
to stop and ask the user — not to fill the gap with a plausible-sounding
guess.

**Before Step 1:** confirm you have already run the project-conventions
check described in `SKILL.md` → "Cross-capability invariants". A TDD that
ignores `CLAUDE.md` is not principal-level work. If you haven't done it
yet, go back and do it now.

### Step 1 — Require a PRD as input (hard gate — refuse to proceed without one)

This is the most important rule in the workflow, and the one the skill most
often gets wrong. **A TDD is a design *for* a stated problem. Without a PRD,
you would be inventing the problem statement and then designing for your own
invention. That is freelancing, not principal-level work. The skill refuses.**

Before classifying the workload, before asking triage questions, before
anything else: confirm a PRD is in hand. If it is not, stop and ask for one.

#### What counts as a PRD

A document the user produced or pointed you at, with enough content to
design against. Acceptable forms:

- Pasted PRD content in the conversation.
- A local file path you can `Read`.
- A public URL you can `WebFetch`.
- A Drive / Confluence / Jira link the user has confirmed you can reach
  (e.g. via MCP).
- An attached / exported document.

#### What does NOT count as a PRD

The skill refuses to draft from any of these:

- A one- or two-line feature description in the prompt
  ("design the new referral feature", "we need a webhook for Stripe").
- A short bulleted wishlist typed into chat instead of a real PRD.
- A verbal summary the user gives in lieu of writing the PRD down.
- Anything you would have to substantially infer or invent to fill out.
- Codebase pointers alone, with no problem statement attached.

When the user has given only a feature blurb, refuse with this language
(or close to it) and stop:

> I won't draft a TDD without a PRD — designing without one would mean me
> inventing the requirements rather than designing for them, and the result
> wouldn't be principal-level. Please paste the PRD here, give me a file
> path or URL I can fetch, or let me know if you'd like help writing the
> PRD first.

Do not soften this. Do not offer to "draft a starting point based on what
you've shared." Do not propose to "fill in reasonable defaults." Those are
exactly the failure modes this gate exists to prevent.

#### How to resolve PRD sources the user names

| Source they mention                | What to do                                                                                                                                            |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Pasted text / inline               | Use it directly.                                                                                                                                       |
| Local file or directory path       | Read the file(s) with `Read` / list with the appropriate tool.                                                                                         |
| URL (public)                       | Use `WebFetch` if available.                                                                                                                           |
| Google Drive / Google Docs link    | First ask: "do you have a Google Drive MCP configured I should use?" If yes, use it. If no, ask the user to either paste the content, export to local file, or share a path/URL you can fetch. Do **not** assume access. |
| Confluence / Atlassian / Jira      | Same pattern: ask if an Atlassian MCP is configured. If not, ask for pasted content, exported file, or a URL.                                          |
| Codebase context                   | Read the relevant repo files yourself if a path is given. Do not scan the entire codebase blindly — ask the user which modules are in scope.            |

If the user names a PRD source you cannot reach (no MCP configured, dead
URL, missing file, permission error), **stop and ask for a fallback** —
paste, export, or a different path. Do not proceed on what you can guess
from the prompt. A PRD you cannot read is the same as no PRD.

#### Read the PRD end-to-end and acknowledge it

Once you have access, read the PRD in full — not a skim of the first
section. Then, before moving to Step 2, write a one-line acknowledgement
back to the user:

> Read the PRD at `<source>`. Covers `<2–4 short phrases describing what
> the PRD actually contains>`. Moving on to workload classification.

This acknowledgement is the proof that you read the PRD instead of
hallucinating from the prompt. If you cannot honestly summarize what the
PRD contains in this line, you have not read it — go back and read it
before continuing.

**Refuse to advance to Step 2 until both conditions hold:** a real PRD is
in hand *and* you have read it end-to-end.

### Step 2 — Classify the workload

Pick **one** primary category. If the system spans more than one (e.g. an API
that also runs a daily reconciliation job), produce **one TDD per workload**
or call out a "multiple workloads" decision explicitly and pick the dominant
one for structure. Categories:

| Category                                 | Use when                                                                                          | Reference file                                  |
|------------------------------------------|---------------------------------------------------------------------------------------------------|-------------------------------------------------|
| CLI utility                              | A command-line tool, one-shot invocation, no long-running process.                                | `references/tdd-cli-utility.md`                  |
| API serving                              | A request/response service (REST, GraphQL, gRPC) with synchronous clients.                        | `references/tdd-api-serving.md`                  |
| Data processing (batch / ETL)            | A pipeline that reads a bounded dataset, transforms it, writes results. Usually long-running.     | `references/tdd-data-processing.md`              |
| Scheduled pull-based processing          | A cron-driven job that periodically pulls from a source and acts on changes.                      | `references/tdd-scheduled-pull.md`               |
| Event-driven / streaming consumer        | A long-running consumer of Kafka / Kinesis / SQS / PubSub / NATS, push-from-broker semantics.     | `references/tdd-event-driven.md`                 |
| Async worker / job-queue consumer        | Worker pool consuming tasks from a queue (Celery / Sidekiq / BullMQ / SQS jobs).                  | `references/tdd-async-worker.md`                 |
| Webhook receiver                         | Push-based HTTP endpoints from third parties (Stripe, GitHub, payment providers, SaaS callbacks). | `references/tdd-webhook-receiver.md`             |
| Library / internal SDK                   | Code that other services consume; no runtime of its own.                                          | `references/tdd-library-sdk.md`                  |

If the workload genuinely doesn't fit any of these, say so and propose a
structure rather than forcing a misfit.

### Step 3 — Triage: confirm the design-changing inputs

Read `references/challenge-prd.md` to remind yourself of the principal-level
review questions.

Every TDD needs an explicit answer — sourced from the PRD or from the user
— for each of these **design-changing dimensions** before drafting:

1. **Load profile** — expected QPS / volume / data size at launch, and the
   1-year horizon.
2. **Latency budget** — p50 / p95 / p99 target, or "best-effort, no SLO" if
   that's the accepted answer.
3. **Consistency requirement** — strong / read-your-writes / eventual (and
   what "eventual" means in latency terms).
4. **Auth model** — who calls this, with what credentials, in what trust
   boundary.
5. **Runtime / infra constraints** — required cloud, K8s, on-prem, language,
   banned dependencies.
6. **One workload-specific dimension** from the category reference — e.g.
   delivery semantics for event-driven, idempotency-under-partial-failure
   for batch, STDIN/pipe behavior for CLI.

For each dimension: if the PRD answers it, record the answer and move on; if
the PRD is silent, ask the user. Batch the unanswered ones into a single
message — do not drip-feed questions.

These six are the dimensions that flip the design itself: DB choice,
caching, sync vs async, multi-region, framework. Skipping any one of them
and proceeding produces a TDD that has to be redone.

Anything outside these six lives in **Open Questions** in the TDD body, not
in the triage batch. Triage is for design-blockers, not for completeness.

If you ask a triage batch, **wait for the answers** before drafting. Do not
proceed against partial answers.

### Step 4 — Read the category reference + cross-cutting NFRs

Before writing, read:

1. The category reference file from the table above.
2. `references/nfrs-checklist.md`.

The category reference tells you which sections are mandatory for that
workload type and which NFRs deserve more depth.

Every NFR in `nfrs-checklist.md` must appear, by name, in section 6 of the
TDD — either with a real target / how-met / how-verified entry, or with an
explicit one-line **N/A — `<reason>`** (e.g. "N/A — internal CLI, no uptime
SLO"). Silent omissions are forbidden: they hide whether the NFR was
considered and rejected, or simply forgotten, and the reviewer cannot tell
which. The discipline is "decide, then write the decision down" — not "drop
what doesn't seem relevant."

### Step 5 — Draft the TDD

Use the common skeleton below, then add or expand sections per the category
reference. Write at principal level: name specific technologies, name
specific tradeoffs, name specific failure modes. Avoid vague phrases like
"highly scalable" or "robust error handling" — replace them with numbers,
mechanisms, or concrete alternatives considered and rejected.

#### Common TDD skeleton (applies to every category)

```markdown
# TDD: <Feature / system name>

**Author:** <tech lead (you, on behalf of the user)>
**Status:** Draft
**Date:** <today>
**Related PRD:** <link or "pasted inline">

## 0. Design constraints from project conventions
- List only constraints that came from `CLAUDE.md` / `CONTRIBUTING.md` /
  `ARCHITECTURE.md` / similar repo-level convention docs (cite the source
  document for each).
- Cover at minimum, when relevant: required base classes / abstractions,
  banned or required dependencies, naming / file-layout patterns, test
  framework + style, definition-of-done checklist, registration steps,
  lint / style rules.
- Note any constraint that *overrides* a default this skill would otherwise
  recommend, so the reviewer can see the deviation is deliberate.
- **Omit this entire section** if no project convention documents exist
  (greenfield design). Do not invent constraints.

## 1. Problem & context
- What user / business problem this solves, in 3–5 lines.
- Who the consumers are and how they reach this system.
- Why now (driver / deadline / dependency).

## 2. Scope
- In scope: bullet list of what this design covers.
- Out of scope: bullet list of what is explicitly NOT covered (and why).

## 3. High-level approach
- The one-paragraph summary a reviewer should walk away with.
- Diagram description in prose (boxes + arrows). If a real diagram would help
  the reader, say so and describe it precisely enough that someone could draw
  it; do not fabricate diagram links.

## 4. Detailed design
- The category-specific sections (see the category reference).
- Data model / schema (or `N/A — <reason>`, e.g. "stateless").
- Key interfaces / APIs / message contracts.
- Concurrency & state model.
- External dependencies and what happens if each one is down.

## 5. Alternatives considered
- At least 2 alternatives. For each: what it was, why rejected.
- Include "do nothing" when there is a credible status-quo option.

## 6. Non-functional requirements
- Walk `references/nfrs-checklist.md` in order. For each NFR, write one of:
  - **Target** (number or SLO) + **how met** + **how verified**, when it
    matters for this workload, or
  - **N/A — `<one-line reason>`** when it does not (e.g. "N/A — internal
    CLI, no uptime SLO").
- Every NFR in the checklist must appear in this section. Omission is a
  decision; the section is where the decision becomes visible.

## 7. Risks & failure modes
- What can go wrong? For each: likelihood, blast radius, mitigation, recovery.

## 8. Rollout & migration
- Phasing (behind a flag? shadow mode? gradual percentage?).
- Backwards compatibility / data migration (or `N/A — <reason>`).
- Rollback story — what does "undo" look like?

## 9. Observability & operations
- Metrics, logs, traces, alerts to add. Be specific (names, dimensions, SLO thresholds).
- Runbook items / on-call surface area.

## 10. Open questions
- Numbered list. Each item should be a real decision the team needs to make,
  with the options and a (tentative) recommendation where you have one.
```

The category reference may add or rename sections (e.g. CLI utility has no
"observability & operations" in the service sense — replace with "exit codes
& diagnostics"). Adapt rather than mechanically repeat.

### Step 6 — Write the file

Write the TDD to **`./TDD-<feature-slug>.md`** in the current working
directory using the `Write` tool. Use a short kebab-case slug derived from
the feature name (e.g. `TDD-billing-webhook.md`).

If a file with that name already exists, ask the user before overwriting:
"there is already a `TDD-x.md` here — overwrite, append, or write to a new
filename?"

After writing, give the user a short summary in chat:
- One-sentence description of what you designed.
- Category used.
- Number of open questions.
- Anything you want them to look at first.

## Style guide for the TDD

- **Concrete over generic.** "Postgres 16 with logical replication to a
  read-replica" beats "scalable relational database".
- **Numbers over adjectives.** "p99 < 200ms at 500 RPS" beats "low latency".
- **Justified, not asserted.** Every meaningful choice has a *why* and what
  it costs.
- **Honest about what you don't know.** Open questions are a feature, not a
  failure of nerve.
- **No invented references.** Don't link to runbooks, dashboards, or repos
  that you have not actually seen. If a link belongs there, write a
  placeholder like `<link to runbook — to be added>`.
- **No emojis** in the TDD body unless the user explicitly asks.
