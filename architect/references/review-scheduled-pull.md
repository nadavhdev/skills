# Architect review aspects — scheduled pull (cron / periodic)

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-scheduled-pull.md` first — that
is the standard this TDD was (or should have been) written against. Use its
required sections, NFR applicability, and pitfalls as the baseline checklist.
This file adds the architect lens.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **Cursor / watermark model.** How the job knows where it left off. Get this
  shape wrong and every run either re-pulls everything or silently skips data;
  changing it later usually means a backfill.
- **Dedup key.** What makes a pulled item unique. Foundational to idempotent
  writes; hard to change after data exists.

## Type-specific hot spots (review questions)
- **Overlap prevention.** If a run takes longer than the interval, can two runs
  execute at once and corrupt the cursor or double-write? A lock / single-flight
  guard is mandatory — absent → `risk`/`concurrency` finding.
- **Watermark persistence & resume.** Does the cursor survive a crash mid-run so
  the next run resumes correctly rather than re-pulling or skipping?
- **Missed-run handling.** After downtime, does it catch up or skip? Is that the
  right choice for the data?
- **Source rate limits / quotas.** Especially for many sources at different
  cadences — does the design respect each source's limits (`ownership`/`risk`)?
- **Idempotent writes + freshness SLO.** Are writes safe under retry, and is
  there a stale-data alert when a source stops updating?
- **Secret storage & rotation** for source credentials.

## Over-engineering & simplification tells
- A streaming consumer / broker for data that changes on a slow periodic cadence
  → point at the simpler polling shape (`over-engineering`).
- A distributed scheduler for a single cron job.
- Per-source bespoke code where one adapter pattern would unify them (or, the
  reverse: a forced abstraction over sources that are genuinely different — judge
  honestly).

## Calibration — NFRs
- **Demand if absent:** overlap lock, watermark/resume, freshness alert,
  per-source rate-limit awareness, idempotent writes, **empty-run / API-quota
  cost awareness** (each tick costs even when nothing changed), and a **test
  strategy** covering the overlap, missed-run, and DST cases.
- **Flag as over-reach if present:** sub-second latency SLOs, real-time delivery
  guarantees — a periodic pull's contract is freshness, not latency.
