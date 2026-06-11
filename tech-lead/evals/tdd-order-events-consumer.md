# TDD: Order events consumer & projections

**Author:** Tech lead (drafted on behalf of the orders team)
**Status:** Draft
**Date:** 2026-06-08
**Related PRD:** `evals/prd-order-events-consumer.md`

## 1. Problem & context

The orders service publishes `order.*` events (created, paid, shipped,
cancelled, refunded) to a Kafka topic partitioned by `order_id`. Several
downstream teams currently consume the raw stream and each rebuild a
local projection of order state. They keep getting bitten by
out-of-order events and inconsistent state across consumers. This TDD
designs a single canonical projections service that owns the materialized
view and exposes a read API.

## 2. Scope

**In scope:** consumer of `order.*` events, idempotent projection writes,
back-fill / replay support, a synchronous read API for the projection,
DLQ for poison events.

**Out of scope:** changes to producer schemas (owned by orders team),
historical analytics queries (handled by the existing data warehouse),
cross-region replication of projections.

## 3. High-level approach

A long-running consumer service (`order-projections`) reads from the
`orders.events.v2` Kafka topic with a stable consumer group. For each
event, it applies a transition to the materialized state in Postgres
inside a single transaction that also stores the event's `(partition,
offset)` for idempotency. The same service exposes a read API
(`GET /orders/{order_id}`) over the projection.

Delivery semantics: at-least-once from Kafka + idempotent application
keyed on `(partition, offset)` → **effectively-once** projection. We do
not need exactly-once delivery primitives from the broker.

## 4. Detailed design

### 4.1 Consumer model

- Consumer group `order-projections-v1`, one consumer per partition,
  partitions sized to current peak (16 partitions on the topic today).
- Manual offset commit after the projection transaction commits.
- Auto-rebalance on deploy; no static assignment.

### 4.2 Projection state (Postgres)

`order_projections(order_id PK, status, total_cents, customer_id,
last_event_partition, last_event_offset, last_event_ts, updated_at)`

`processed_events(partition, offset, order_id, applied_at PK(partition,
offset))` — idempotency log; lookup-before-apply on `(partition, offset)`
inside the transaction.

### 4.3 Ordering & out-of-order handling

Kafka guarantees order within a partition, and partitioning is by
`order_id`, so events for one order arrive in order *as written by the
producer*. However, producers occasionally re-emit historical events
during their own replays. We protect against stale overwrites by
comparing the incoming event's `event_seq` (producer-side monotonic
sequence per order) against `order_projections.last_event_seq`; reject
older events.

### 4.4 DLQ for poison events

Events that fail schema validation, fail to deserialize, or trigger an
invalid state transition (e.g. `shipped` arriving on an order in
`cancelled`) go to a DLQ topic `orders.events.v2.dlq` after 3 in-process
retries. The consumer continues; the bad event does not block the
partition.

### 4.5 Read API

`GET /orders/{order_id}` returns the current projection. Latency-sensitive
callers can subscribe to a separate `order.projection.updated` topic we
emit (out of scope here, but the design allows it).

## 5. Alternatives considered

- **DynamoDB instead of Postgres for the projection store.** Rejected for
  this workload: 50K orders/day at peak, complex secondary access
  (customer lookups, status filters) — Postgres serves it cheaper with
  joins and proper indexes. We'd revisit at 10× scale.
- **Exactly-once via Kafka transactions + Postgres outbox.** Rejected as
  premature: at-least-once + idempotent writes give us effectively-once
  at much lower operational surface.
- **Per-team local projections (status quo).** Rejected — the inconsistency
  pain across teams is the explicit driver for this TDD.

## 6. Non-functional requirements

- **Throughput.** Sustained 200 events/sec, peak 1000/sec (flash
  promo / refund storm). *How met:* 16-partition fan-out, one consumer
  per partition.
- **Lag SLO.** p95 lag < 30s. *Verified:* Kafka consumer-lag metric.
- **Read latency.** Projection read p99 < 100ms. *How met:* Postgres
  with PK lookup, connection pooling.
- **Availability.** 99.9% for the read API.
- **Durability.** Postgres PITR (7-day window). DLQ topic retention
  14 days.
- **Security.** Mutual TLS to Kafka, internal-only read API.
- **Cost.** Fits in existing data-platform Postgres tier.
- **Compliance.** PII (customer_id) in the projection; access controlled
  by existing internal-API auth.
- **DR.** Single-region. On region loss, rebuild projections by replaying
  from earliest Kafka retention (7 days) into a fresh cluster.

## 7. Risks & failure modes

- **Schema evolution breaks consumer.** Mitigation: Avro schemas
  registered, BACKWARD-compatibility checked in CI. Any breaking change
  forces a new topic version.
- **Projection drift after partial outage.** Mitigation: replay tool
  (see §8) can re-apply from any offset; idempotency log prevents
  duplicate application.
- **DLQ growth.** Risk: producer bug floods DLQ. Mitigation: alert on
  DLQ rate > 0.1% of input rate for 10 min.
- **Postgres saturation under back-fill.** Mitigation: back-fill runs
  on a dedicated consumer with throttled writes.

## 8. Rollout & migration

- Deploy in shadow mode: consumer reads but doesn't expose the read API.
  Compare projection state to the existing per-team projections for one
  week.
- Cut over read API behind feature flag `projection_api_enabled`,
  per-consumer-team opt-in.
- Back-fill tool: replay topic from earliest retention into a fresh
  consumer group; required at first cutover and for DR drills.
- Rollback: flip flag, consumers fall back to local projections (which
  we don't delete until 30 days post-cutover).

## 9. Observability & operations

- Metrics: `events_consumed_total{result}`, `projection_lag_seconds`,
  `dlq_emitted_total`, `read_api_latency_seconds`,
  `consumer_rebalance_total`.
- Logs: structured, include `order_id`, `partition`, `offset`,
  `event_seq`.
- Alerts: lag > 30s for 5 min; DLQ rate > 0.1% for 10 min;
  consumer rebalance storms (> 3 in 5 min); read API error rate > 1%.
- Runbook entries: cold-start replay, DLQ inspection + redrive,
  partition reassignment.

## 10. Open questions

1. **Replay behavior under schema migrations.** When we cut a v3 topic
   for a breaking schema change, how do we replay v2 events into the
   new projection? Recommend a one-shot translator job; need orders-team
   sign-off on the schema mapping.
2. **Per-customer read pattern frequency.** PRD doesn't say whether
   customer-level lookups (find all of customer X's orders) need to be
   sub-100ms. If yes, we need a secondary index on `customer_id`; if
   "best effort", we don't. Affects the schema task scope.
