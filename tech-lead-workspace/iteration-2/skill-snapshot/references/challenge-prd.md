# Challenging a PRD — principal-level questions

PRDs are written by product. They usually describe **what** users will see
and **why** it matters to the business. They are usually thin on the parts
that drive engineering cost and risk. Your job as tech lead is to make the
missing engineering-relevant information explicit — before drafting, if it
flips the design; in the **Open Questions** section, otherwise.

Use this list as a thinking aid. **Do not paste it all into the conversation
or the TDD** — pick the questions that matter for the workload.

## 1. The problem itself

- What is the actual user / business outcome? (Not "build feature X" —
  what is X *for*?)
- What metric moves when this ships? How will success be measured?
- Why now? What changes if this slips by a quarter?
- What is the smallest version of this that still moves the metric?
  (Push back on scope where possible.)
- Are we solving the right problem? Is there a non-engineering solution
  (process change, vendor, manual workflow) that's cheaper?

## 2. Users, load, and access

- Who calls this, from where? Public internet, mobile clients, internal
  services, employees, third parties?
- How many users / clients / tenants on day 1, in 6 months, in 2 years?
- Expected request volume — peak and steady-state. Bursty or smooth?
- Geographic distribution. Single region or multi-region?
- Auth model. Identified user? Service-to-service? Anonymous public?

## 3. Data

- What data does this read / write / own?
- Who else writes to that data? Is there a single writer, or contention?
- What's the data classification — PII, payment, health, secrets, public?
- What's the volume — rows now, growth rate, retention?
- What's the consistency requirement — strong, read-your-writes,
  eventual, "doesn't matter"?
- What's the source of truth? If the new system caches or derives data,
  what happens on divergence?

## 4. Performance & latency

- What latency does the caller expect? Tied to a user-visible interaction
  or async / background?
- What's the budget for downstream calls (DB query, external API)?
- What's the acceptable degradation mode under load (slow, error, drop,
  queue)?

## 5. Failure modes

- What's the worst that can happen if this is wrong / down / slow?
- What's the blast radius — single user, single tenant, whole org, all
  customers?
- What does "down" cost — revenue, reputation, regulatory exposure, life
  safety?
- If a downstream dependency is down for 1 hour, what should the user
  experience? What should the data look like when it comes back?

## 6. Idempotency & duplicates

- Can the same request arrive twice? (Almost always: yes.)
- What should happen — succeed once, fail the duplicate, return cached
  result, log and ignore?
- For writes: is there a natural idempotency key (order ID, event ID,
  request ID), or do we need to fabricate one?

## 7. Ownership & operations

- Which team owns this in production? Who is on-call?
- What's the deployment cadence — every commit, weekly, gated?
- What does "rollback" look like for this? In particular: what about data
  changes that can't be rolled back?
- Where are logs / metrics / traces aggregated? Is the on-call team going
  to be able to debug this with what we expose?

## 8. Constraints & non-negotiables

- Cloud / region constraints (regulatory, contractual).
- Language / framework constraints from the team or org.
- Compliance regimes — GDPR, HIPAA, SOC2, PCI-DSS. Which apply?
- Budget ceiling. Per-month cost the design has to fit under.
- Hard deadlines — and what slips if we don't make them.

## 9. Integration points

- What systems does this need to talk to? Who owns each of them?
- Are those systems stable, or actively changing? Is the contract documented
  and versioned?
- Are there rate limits / quotas / SLAs we have to respect on those calls?

## 10. The unstated assumptions

- What is the PRD assuming about the existing system that may not be
  true? (E.g. "users have an account" — do all users? Anonymous flow?)
- What is the PRD assuming about *another* team's roadmap? Is that
  team aware?
- Where in the PRD is a sentence like "this should just…" hiding a
  multi-month engineering effort?

## How to use this in triage

Walk down the list mentally. For each question, classify:

- **Critical to design** — answer flips the architecture, the tech, or
  the team needed. Ask the user **before** drafting.
- **Important but not blocking** — answer affects implementation detail
  or operational story. Put in **Open Questions** in the TDD.
- **Not applicable** — skip silently.

**Be ruthless about the "critical" bucket — keep it to 3–6 questions per
upfront batch.** Eleven questions in chat is annoying and signals you're
trying to outsource the design. Ask the few that genuinely change the design
upfront, and put the rest in the document.
