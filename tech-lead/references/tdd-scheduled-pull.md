# TDD reference — scheduled pull-based processing

## When this fits

A cron-driven (or scheduler-driven) job that **periodically pulls from a
source and acts on changes**. Source could be an external API, an inbox, an
S3 prefix, a database table polled for "what's new since last run".
Examples: poll a vendor API every 5 minutes for new orders; check an S3
landing zone every hour and ingest new files; reconcile two systems nightly;
poll an SFTP drop for new files.

**Not this** if: the source pushes to you (see webhook or event-driven), the
job operates on a known bounded input (see data processing), or it's a
queue-triggered worker (see async worker).

## What's different about scheduled pull

- **The schedule is the trigger** — and the schedule has subtle traps:
  overlapping runs, missed runs, clock skew, DST.
- **State across runs matters.** "What's new since last time" requires a
  cursor / watermark / high-water-mark stored somewhere durable.
- **The source might lie.** External APIs can replay, return out-of-order,
  rate-limit, or temporarily lose data. Your code is the only thing standing
  between source weirdness and your downstream.
- **Polling has a cost.** Each tick consumes API quota, money, and
  capacity even when nothing changed. Empty-run cost matters.

## Required sections (in addition to the common skeleton)

### 4a. Schedule & trigger

- Cadence — every N minutes / hourly / daily. Why this cadence (latency
  requirement, source rate limit, cost).
- Scheduler — cron, K8s CronJob, Airflow, EventBridge, etc. Why.
- Time zone of the schedule. **DST handling** — runs that fall in the
  spring-forward gap, runs that double-fire in fall-back.

### 4b. Cursor / watermark

- What is the "since" value — last-modified timestamp, monotonically
  increasing ID, ETag, source-provided pagination cursor?
- Where stored — DB row, S3 file, parameter store. Concurrency-safe?
- What does "first run" mean — backfill from epoch? From a configured start?
- What happens if the cursor is lost or corrupted — manual recovery story.

### 4c. Source contract & resilience

- API / source endpoint(s), auth, rate limit, pagination.
- What "no new data" looks like vs "API is down" — these must not be
  confused.
- Retry policy when the source is down (exponential backoff, jitter, cap).
- Behavior on auth failure (alert immediately — usually a credential
  rotation issue).
- What if the source replays data we've already processed — dedup story.

### 4d. Overlap & concurrency

- Can two runs overlap if one is slow? **Default answer should be NO** for
  most pull jobs (causes duplicate processing, race on cursor).
- Mechanism to prevent — distributed lock (DB row, Redis), advisory lock,
  "skip if previous still running" flag on the scheduler.
- What happens if a run is missed (scheduler down, locked out by prior
  long run) — catch up next tick, or trigger a manual catch-up?

### 4e. Processing model

- Per-record vs batch processing of pulled items.
- Idempotency — if a record was partially processed and the run died, can
  the next run safely re-process it?
- Side effects on downstream systems — same idempotency rules as
  data processing.

### 4f. Backpressure / large delta

- What if the source has a 10× larger delta than usual (e.g. recovery
  after a source outage)? Does this run try to process everything in one
  shot, or chunked across multiple runs?
- Memory / time budget per run, and how exceeding it is handled.

## NFRs that matter for scheduled pull

From `nfrs-checklist.md`:

- **Reliability** — "every Nth run succeeds" SLO. Freshness SLO ("data is
  at most M minutes stale").
- **Observability** — record count per run, run duration, last-success
  timestamp, cursor lag. Alerts on freshness breach and on N consecutive
  failures.
- **Cost** — API quota usage, infra cost per N runs. Empty-run cost.
- **Security** — credential storage and rotation for the source API.

Often omitted: latency (not the right frame — use freshness), multi-region
(unless source allows it).

## Common pitfalls

- **No lock — two runs overlap and corrupt the cursor.**
- **Cursor advances before processing succeeds** — records lost on failure.
- **Cursor advances after processing succeeds, but processing isn't
  idempotent** — partial-failure re-runs duplicate side effects.
- **"No data" confused with "API down"** — silently swallows incidents.
- **No alert on stale cursor** — the job has been broken for a week and
  no one noticed because runs return "success: 0 records".
- **DST-related double or missed runs** untested.
- **No way to manually replay** a specific window after a downstream bug.
- **Per-tick API call even when nothing changed** — burns quota / money.
- **Schedule cadence too tight for the source's rate limit** — perpetual
  429s.

## Key decisions to surface in "Alternatives considered"

- **Pull vs push** — does the source actually support webhooks / events?
  If yes, pull may be the wrong pattern (see webhook receiver / event-driven).
- Cursor type (timestamp vs ID vs ETag) — and how each handles
  out-of-order updates.
- Locking mechanism.
- Per-tick processing budget — fixed window vs catch-up-everything.
- Scheduler choice (cron in K8s CronJob, Airflow, cloud-native).
