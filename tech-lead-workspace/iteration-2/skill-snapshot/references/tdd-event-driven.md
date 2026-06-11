# TDD reference — event-driven / streaming consumer

## When this fits

A long-running consumer of an event log or message broker — Kafka, Kinesis,
SQS, Google Pub/Sub, Azure Service Bus, NATS, RabbitMQ. The broker pushes /
the consumer pulls in a tight loop; events are unbounded (in time, in
volume); state often persists across messages. Examples: CDC consumer that
projects into a read model, event-sourced workflow runner, real-time fan-out
to downstream services, fraud-detection scoring pipeline.

**Not this** if: the input is bounded (see data processing), the trigger is
a clock (see scheduled pull), or the work-item is a one-shot task to do
(see async worker — though the lines blur; the test is whether you're
processing a stream of domain events or a stream of work-to-do).

## What's different about event-driven consumers

- **Delivery semantics are everything.** At-most-once, at-least-once,
  effectively-once — these are not interchangeable. "Exactly-once" usually
  means at-least-once + idempotent processing.
- **Ordering is a constraint.** Per-partition ordering is normal; global
  ordering is rare. Your partition key choice determines what can be
  parallelized.
- **Lag is the primary health signal.** Not RPS, not latency — consumer lag.
- **Poison messages can stop the world.** A single un-processable message in
  an ordered partition blocks everything behind it. DLQ design is mandatory.
- **Rebalancing has cost.** Adding / removing consumer instances pauses
  processing briefly and can re-deliver in-flight work.

## Required sections (in addition to the common skeleton)

### 4a. Broker & topic model

- Broker (Kafka / Kinesis / SQS / PubSub / …) and why this one.
- Topic(s) consumed, partition count, retention, expected throughput
  per topic.
- Producer-side contract — who produces, schema (Avro / Protobuf / JSON
  schema), schema-evolution rules.
- Consumer group / subscription name. Single-consumer-group or fan-out?

### 4b. Delivery semantics & ordering

State explicitly:

- **Delivery guarantee from broker** — at-most-once, at-least-once,
  exactly-once (rare; brokers like Kafka offer it with strict caveats).
- **Ordering guarantee** — per-partition ordering is typical. State the
  partition key and what that key groups (per-user, per-tenant, per-entity).
- **End-to-end guarantee you provide** — usually at-least-once + idempotent
  effects = effectively-once. Name what makes effects idempotent (idempotency
  key, conditional write, dedup table with TTL).

### 4c. Offset / checkpoint management

- When are offsets committed — before processing? After? After downstream
  write?
- Commit cadence — per message (high cost, exact replay) vs per batch
  (cheap, replays N messages on failure).
- What replay looks like — restart from earliest? From committed?
- How to recover from a poisoned commit (offset advanced past a still-
  unprocessed message).

### 4d. Poison-message / DLQ handling

- **Trigger** — N retries with backoff, or specific exception types, or
  per-message timeout exceeded.
- **DLQ destination** — separate topic / queue, with the original payload,
  the failure reason, the consumer attempt count.
- **Replay path** — who looks at DLQs, how messages are corrected and
  re-injected.
- **Alarm threshold** — DLQ count > X triggers a page.

### 4e. Concurrency & parallelism

- Instances × workers-per-instance. Bounded by what (partition count, DB
  connections, downstream rate limit)?
- In-flight message count per worker.
- Per-partition serialization — multiple partitions in parallel, but
  one message at a time per partition (typical for ordered processing).

### 4f. Back-pressure

- What's the slowest downstream — the DB, an external API, another topic?
- What happens when the consumer can't keep up — lag grows, broker
  retention is the real ceiling. State the retention and what "X minutes
  of lag = data loss" means.
- Rate-limit-aware processing if a downstream caps you.

### 4g. Stateful processing (if applicable)

- Local state (windowed aggregations, joins) — where stored (RocksDB,
  in-memory, external Redis)? How recovered on restart?
- Checkpointing of state, independent of offset commit.

### 4h. Schema evolution

- Compatibility mode (backwards / forwards / full / none) on the registry.
- How a new field is added without breaking older consumers / producers.
- How a breaking change is rolled out (new topic? versioned message
  envelope?).

## NFRs that matter for event-driven

From `nfrs-checklist.md`:

- **Reliability** — message loss budget (usually zero). DLQ rate budget.
- **Performance** — consumer lag SLO ("p95 lag < 30s"), throughput
  (messages/sec).
- **Observability** — lag per partition, processing rate, DLQ rate, retry
  rate, downstream-error rate. **Alerts on lag growth** are mandatory.
- **Resilience** — downstream failure containment, partial poison-message
  tolerance.
- **Scalability** — partition count headroom for future scale-out.

Often less central: end-to-end latency from producer to consumer is usually
fine to call out but the lag SLO captures it.

## Common pitfalls

- **Committing offsets before processing** — at-most-once by accident, lost
  messages on crash.
- **Long-running per-message work** that exceeds the broker's session
  timeout — consumer is kicked, rebalance, message redelivered, repeat.
- **No DLQ** — a single bad message stops the partition forever.
- **Idempotency declared but not implemented** — "we'll just use the
  message ID" with no dedup store.
- **Schema change without compatibility mode** — old consumers crash on
  unknown fields, or new consumers crash on missing fields.
- **One consumer instance per partition with no headroom** — single-pod
  failure halves throughput and lag explodes.
- **Stateful state lost on restart** — windowed aggregations restart
  empty, downstream sees discontinuity.
- **Treating lag as a curiosity, not as a primary alert.**

## Key decisions to surface in "Alternatives considered"

- Broker choice (Kafka vs Kinesis vs SQS vs PubSub) — and the operational
  cost of each.
- Partition key (and what it implies about ordering and hot-partition risk).
- Delivery semantics target (at-least-once + idempotent vs aiming for
  exactly-once).
- Where stateful state lives (in-broker, in-RocksDB, in external store).
- Push vs pull (most modern brokers — Kafka, Kinesis — are pull; SQS too;
  but the consumer feels event-driven).
