# TDD reference — async worker / job-queue consumer

## When this fits

A pool of workers consuming **tasks-to-do** from a queue. Triggered by
enqueue events (often from an API that hands off long-running work).
Examples: Celery / Sidekiq / BullMQ / RQ / SQS-jobs / GCP Tasks / cloud
Functions on Pub/Sub jobs. The unit is a "do this thing", not a "this thing
happened".

**Not this** if: you're processing a stream of **domain events** that
multiple subscribers care about (see event-driven), or the trigger is a
clock (see scheduled pull). The line is fuzzy — the test is *intent of the
sender*: "please go do X" → worker; "X happened, you decide what to do
about it" → event-driven.

## What's different about workers

- **The enqueuer expects work to eventually complete.** Unlike streaming,
  the producer often *waits* for an indirect result (status polled by the
  API, callback fired, DB row updated).
- **Latency from enqueue to completion is a real SLO** — different from
  per-message latency in streaming because there's a user/API waiting.
- **Visibility timeouts and lease semantics shape retries.** A worker that
  takes longer than the lease loses the job and a duplicate may run.
- **Tasks are usually heterogeneous.** Different task types have different
  durations and resource profiles, sharing one queue is a noisy-neighbor risk.

## Required sections (in addition to the common skeleton)

### 4a. Queue & enqueue model

- Queue technology (Redis-backed Celery / SQS / Postgres-based / cloud
  Tasks). Why.
- Enqueue contract — who enqueues, payload schema (size limits!), task
  routing keys.
- Priorities / multiple queues if tasks are heterogeneous. State the
  policy that prevents low-priority work from starving important work.

### 4b. Worker pool

- Worker process model — pre-fork, gevent, asyncio, threadpool. Why.
- Concurrency per worker (and per host).
- Auto-scaling rule (CPU? queue depth? lag?). State the scale-up speed
  and the cost ceiling.

### 4c. Task execution

- Max task duration. **Must be < visibility timeout / lease** — otherwise
  double-execution is guaranteed.
- Per-task timeout enforcement — soft (worker raises) vs hard
  (process killed). What happens to in-flight DB transactions if hard-killed?
- Cancellation — can a task be cancelled mid-run? If so, how.

### 4d. Retries & idempotency

- Retry policy — count, backoff, jitter. Per task type, or global.
- What's retried (transient errors) vs not (permanent — bad payload).
- Idempotency — **mandatory** because at-least-once is the default. Either:
  natural idempotency (e.g. "set status to X"), dedup key + dedup store, or
  conditional writes.
- Dead-letter destination after retries exhausted. Who handles DLQ.

### 4e. Result reporting

- Where the result lands — DB row, status endpoint pollable by client,
  webhook callback, separate "results" queue.
- Failure reporting — same channel, with structured failure reason.
- Latency target enqueue → result (the API's SLO).

### 4f. Poison & long-tail tasks

- What stops a single bad task from blocking a worker indefinitely
  (timeout + DLQ).
- What stops a flood of slow tasks from starving fast ones (priority
  queues, separate worker pools, per-tenant quotas).

### 4g. Operations

- How to drain workers for deploys (stop consuming new tasks, finish
  in-flight, then exit).
- How to pause / resume the queue without losing tasks.
- How to manually replay DLQ items.

## NFRs that matter for async workers

From `nfrs-checklist.md`:

- **Reliability** — per-task success rate, DLQ rate budget.
- **Performance** — queue lag (oldest unprocessed task age), task
  throughput, enqueue-to-complete latency.
- **Observability** — queue depth per queue, in-flight task count, task
  duration histogram per task type, retry rate, DLQ count. Alerts on
  queue-depth growth and DLQ rate.
- **Cost** — directly tied to worker pool size; auto-scaling
  cost / floor / ceiling.
- **Resilience** — visibility timeout > p99 task duration with margin;
  graceful shutdown.

Often omitted: deep ordering (most worker queues don't promise order; if
you need it, you're probably actually doing event-driven).

## Common pitfalls

- **Tasks longer than visibility timeout** — the broker re-delivers,
  duplicates run, idempotency wasn't built in, side effects happen twice.
- **Retries without idempotency** — every infra blip doubles the writes.
- **One giant queue, all task types mixed** — one slow type drains the
  pool, fast tasks queue up.
- **No DLQ** — failed tasks loop forever or vanish silently.
- **Payload too big for the queue** — task carries data instead of an ID;
  hits broker payload limit; falls back to "stuff it in S3 and pass a
  pointer" with no plan for cleanup.
- **No graceful shutdown** — deploy kills in-flight tasks, they replay,
  side effects double up.
- **Counting "task succeeded" by "no exception"** — partial-success
  silently masks errors (e.g. caught and logged, but reported success).
- **Result polling that hammers the DB** — clients poll every second for
  hours.

## Key decisions to surface in "Alternatives considered"

- Queue technology (Redis vs SQS vs Postgres vs cloud-native Tasks).
- Single queue with priorities vs multiple queues with dedicated workers.
- Sync (API blocks until done) vs async (API returns task ID) — and the
  resulting result-reporting channel.
- Visibility timeout choice — and the corresponding hard task timeout.
- Push (worker invoked by broker / function) vs pull (worker long-polls).
