# Architect review aspects — async worker (queue-driven task pool)

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-async-worker.md` first — that is
the standard this TDD was (or should have been) written against. Use its required
sections, NFR applicability, and pitfalls as the baseline checklist. This file
adds the architect lens.

The line between this and event-driven blurs: the test is whether the consumer
processes a stream of **work-to-do** (async worker) vs a stream of **domain
events** (event-driven). If the design is really the latter, say so.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **Job/message schema.** What a job carries. Consumers and producers couple to
  it; versioning it later is the same problem as any contract.
- **Idempotency key.** What makes a job safe to run twice. Foundational — queues
  redeliver.
- **Retry / visibility semantics.** Max attempts, backoff, visibility timeout vs
  job duration. Shapes correctness under failure.

## Type-specific hot spots (review questions)
- **Idempotency under retry.** Jobs *will* be redelivered. Is each job safe to
  execute more than once, or will retries double-charge / double-send
  (`data`/`risk`)?
- **Retry + backoff + max-attempts + DLQ.** Is there a bounded retry policy and a
  dead-letter destination for jobs that exhaust it? Unbounded retry of a poison
  job is a `risk` finding.
- **Visibility timeout vs job duration.** If a job runs longer than the
  visibility timeout, it gets redelivered while still running → duplicate work.
  Is the timeout sized to the work?
- **Concurrency bound.** Workers × instances bounded by DB connections /
  downstream rate limits? Unbounded fan-out is a `scalability`/`risk` finding.
- **Ordering assumptions.** Queues generally don't guarantee order — does the
  design quietly assume it?

## Over-engineering & simplification tells
- A full workflow / saga engine for simple fire-and-forget tasks
  (`over-engineering`).
- Chasing exactly-once where idempotent at-least-once is simpler.
- A new queue technology where the codebase already runs one (`codebase-fit`).

## Calibration — NFRs
- **Demand if absent:** idempotency, bounded retry + DLQ, visibility/timeout
  sizing, concurrency bound, **worker-pool cost** (autoscale floor/ceiling — cost
  scales with pool size), **graceful drain on deploy**, and a **test strategy**
  covering retry, duplicate-delivery, and timeout cases.
- **Flag as over-reach if present:** strict ordering guarantees or exactly-once
  where the work doesn't require them.
