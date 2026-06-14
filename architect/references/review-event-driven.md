# Architect review aspects — event-driven / streaming consumer

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-event-driven.md` first — that is
the standard this TDD was (or should have been) written against. Its "What's
different", "Required sections", "Common pitfalls", and NFR list are the baseline
checklist the design must satisfy. This file adds the architect lens.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **Partition key.** Determines ordering and what can be parallelized, and seeds
  hot-partition risk. Re-keying a live topic is a migration. Is the key right for
  the PRD's ordering needs?
- **Delivery-semantics target.** At-least-once + idempotent effects vs chasing
  exactly-once. The choice shapes the whole design.
- **Schema-evolution compatibility mode.** Backwards/forwards/full — locks how
  producers and consumers can change independently.
- **Where stateful state lives** (in-broker / RocksDB / external store), if the
  consumer is stateful.

## Type-specific hot spots (review questions)
- **Idempotency actually implemented.** "We'll use the message ID" with no dedup
  store is the classic gap — at-least-once delivery makes duplicate effects
  inevitable. Absent dedup → `data`/`risk` finding.
- **Offset commit ordering.** Committed *after* the downstream write, not before?
  Commit-before-processing is at-most-once by accident → lost messages on crash
  (`blocker`-grade `data`).
- **DLQ / poison-message handling.** A single bad message in an ordered partition
  blocks everything behind it. No DLQ → `risk` finding, often `blocker`.
- **Consumer lag SLO + alert.** Lag is the primary health signal. Is it
  monitored and alerted, not treated as a curiosity (`observability`)?
- **Back-pressure & retention ceiling.** What happens when the consumer can't
  keep up — and does "X minutes of lag = data loss" against broker retention?
- **Rebalance / session-timeout vs per-message work** — long work gets the
  consumer kicked and redelivered.

## Over-engineering & simplification tells
- Chasing **exactly-once** when at-least-once + idempotency is simpler and
  sufficient → point at the simpler target (`simplification`).
- Stateful stream processing where stateless projection would do.
- Kafka / a heavy broker where an existing queue (SQS, the codebase's current
  broker) satisfies the PRD (`codebase-fit`/`over-engineering`).

## Calibration — NFRs
- **Demand if absent:** delivery semantics + idempotency, DLQ, offset strategy,
  lag SLO + alert, schema-compatibility mode, a **consumer/schema rollout +
  rollback plan** (how a new consumer or schema version ships without breaking
  in-flight processing), and a **test strategy** covering duplicate, out-of-order,
  and poison-message cases.
- **Flag as over-reach if present:** exactly-once guarantees sold as free;
  global ordering where per-partition suffices.
