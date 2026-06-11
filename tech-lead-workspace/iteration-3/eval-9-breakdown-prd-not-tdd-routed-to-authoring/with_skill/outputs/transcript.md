# Response

I won't break this down without a TDD — without one, the tasks would be my opinions about a feature, not a decomposition of a design the team has agreed on, and the result wouldn't be principal-level.

I read the file you pointed me at (`/Users/nhoze/.claude/skills/tech-lead/evals/prd-referral-api.md`) and it is a PRD: it states the problem, the goals, user stories, and constraints, but it does not state the design. A task breakdown decomposes a *design*, and there isn't one here yet.

Looking at what the PRD leaves unspecified, the design decisions a TDD would have to nail down before any breakdown is meaningful include:

- Code generation and uniqueness strategy (collision handling at scale for 2.1M MAU, growing 8% MoM).
- Attribution model — how a deep-link install ties back to a referrer, how a manually-entered code is reconciled, what happens when both signals arrive.
- Storage and ownership of the referral graph and the redemption state machine (pending install → activated → credit issued).
- Integration seams with the existing Stripe + Postgres credit ledger, including idempotency on credit issuance.
- Fraud controls (the PM has explicitly flagged this as open; legal is involved).
- Code expiry policy (also flagged open by the PM).
- Service shape — FastAPI service, Go service, or extension of an existing one — and the API contract the mobile team needs by July 1.
- Data pipeline to Looker/Snowflake for the marketing dashboard, including event schema and ownership boundary with the data team.

Each of those is a design call, not a ticket. Picking them silently to generate Jira tickets would lock the team into my guesses.

## What I'd suggest

Two options, pick one:

1. **I author the TDD first.** I'll use the TDD-authoring capability of this same skill, working from this PRD. I'll challenge the PRD's gaps up front, pick the right workload template (this is an API-serving design with an async credit-issuance component), and produce a TDD you can review. Once it's approved, I'll come back and break it into Jira tickets.

2. **You already have a TDD I haven't seen.** If the team has drafted one, paste it, give me a file path, or share a link I can fetch, and I'll go straight to the breakdown.

Which would you like?
