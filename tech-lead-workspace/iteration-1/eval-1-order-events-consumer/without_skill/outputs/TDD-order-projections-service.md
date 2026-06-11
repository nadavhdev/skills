# TDD: Order Projections Service

**Author:** Tech Lead (on behalf of Order Platform team)
**Status:** Draft for engineering review
**Related PRD:** `prd-order-events-consumer.md` (Marcus Webb, 2026-06-02)
**Target GA:** end of Q3 2026 (hard deadline: before holiday ramp)

---

## 1. Summary

Build a **stateless Kotlin (Spring Boot) consumer + read API** that materializes a denormalized per-order projection from the `orders.events.v1` Kafka topic into a low-latency primary store, and serves `GET /orders/{order_id}/projection` behind the existing internal API gateway.

The two real engineering risks — and the bulk of this design — are:

1. **Consumer correctness under at-least-once delivery + apparent out-of-order events.** We treat the projection as a deterministic, idempotent reduction over a per-order event log keyed by `(order_id, sequence)`, with a CAS-style conditional write. We do **not** trust wall-clock or arrival order.
2. **Storage choice on GCP.** The PRD mentions DynamoDB; we are on GCP, so DynamoDB is rejected outright (cross-cloud egress, ops burden, no managed VPC peering story worth taking on for this). The recommendation is **Cloud Spanner** for the primary store, with **Memorystore (Redis)** as an optional read-through cache only if p99 read latency under load proves we need it. Postgres (Cloud SQL) is the documented fallback if Spanner cost is rejected; the design works with either.

NFR targets from the PRD: **read p99 < 50ms**, **event→projection p95 < 2s**, **full backfill** must be safe to run on demand. Volume is modest (~50K orders/day, peak 18K/day burst, ~14 event types per order → low-six-figure events/day steady-state, low-millions for backfill).

---

## 2. Problem statement & scope

### 2.1 What we are building

A new service, `order-projections`, with two responsibilities:

- **Consumer**: subscribe to `orders.events.v1` (24 partitions, Avro/Confluent SR, BACKWARDS compat), fold events into a per-order projection document, persist atomically.
- **Read API**: `GET /orders/{order_id}/projection` behind the existing internal API gateway, replacing the customer-app's 5s polling of the order service.

Plus operational surface:

- A **replay** mechanism to rebuild one order, a list of orders, or the entire store from topic start.
- A **DLQ** for events that violate transition rules.

### 2.2 What we are explicitly **not** building (this milestone)

- Per-customer order list / search projection (called out in PRD as a future projection; we make sure the storage model does not preclude it, but we do not implement it).
- Order analytics / historical aggregations (batch pipeline owns this).
- Replacement of order service as source of truth — projections are derived, disposable, and always rebuildable from the log.
- Write APIs on the projection (read-only).
- Push/streaming delivery to the customer app (HTTP GET only; sub-second poll or SSE can come later if needed — current PRD only asks for fast reads).

### 2.3 Audience for this doc

Order platform engineers (Kotlin + Java), SRE (paging + dashboards), DBA/data-platform (Spanner/Cloud SQL provisioning), and the customer-app team (API contract).

---

## 3. Background and constraints (calibrated)

From the PRD, plus checks:

- **Volume**: 50K orders/day average, ~18K/day Black Friday burst *per the PRD* — note this is the per-day **order count**, not event rate. With ~14 event types per order lifetime and roughly 6–8 actually firing per order, steady-state event rate is **~3–5 events/sec average, ~50–100 events/sec peak** (intra-day spikes during flash sales), and **~3–8M events** in the topic if retention is the default 7 days. This is small. Capacity is not the hard part.
- **Polling RPS**: PRD says 120 RPS from polling. 50K orders/day at 5s polling cannot generate 120 RPS unless customers keep order pages open for tens of minutes; the real shape is bursty around order placement and shipment. We design the read API for **400 RPS sustained, 2,000 RPS short burst** (5x PRD headroom; trivial for any of the candidate stores).
- **Delivery semantics**: at-least-once. Duplicates are guaranteed; the consumer must be idempotent.
- **Ordering**: PRD claims "events arrive out of order across partitions during rebalancing". The topic is keyed by `order_id` and a single key always lands on a single partition; Kafka guarantees per-partition order. The most plausible root causes for what the team has observed are:
  - **Consumer-side parallelism**: events from one partition handed to multiple worker threads → reorder before write.
  - **Rebalance double-delivery**: partition reassignment with in-flight, uncommitted offsets → duplicates that look like out-of-order replays when paired with optimistic, non-conditional writes.
  - **Producer retries without idempotence**: the order service producer may not be idempotent, allowing reorder on retry.
  - **Multi-producer races**: if order service has multiple writers per order, the producer side could publish out of business-logic order even though Kafka preserves arrival order.

  The design must be **correct regardless of which of these is true**, because we cannot fix the producer or the broker from inside this service. Concretely: we treat the input as an **unordered stream with monotonic per-order sequence numbers** and reduce by sequence, not by arrival order. See §6.

- **Platform**: GCP, GKE. Kafka is Confluent on GCP (Pub/Sub-mirrored). Avro/Confluent Schema Registry, BACKWARDS compat enforced — we can deserialize old payloads with a new schema.
- **Languages**: Kotlin + Spring Boot (matches order service team; the consumer benefits from coroutines and Kotlin's sealed classes for event modeling).
- **Auth**: behind existing internal API gateway. The gateway already enforces customer identity → `customer_id`. The projection store must therefore include an `owner_customer_id` for authorization.
- **Deadline**: live by **Sept 30 2026**. ~16 weeks from this doc. See §15.

---

## 4. Goals, non-goals, and NFRs

### 4.1 Functional goals (from PRD)

| # | Requirement |
|---|---|
| F1 | Consume `orders.events.v1`, fold into projection per order. |
| F2 | `GET /orders/{order_id}/projection` returns latest projection. |
| F3 | Invalid state transitions are logged + DLQ'd, do not block the consumer. |
| F4 | Ops can replay a single order to rebuild its projection. |
| F5 | Full backfill is supported. |

### 4.2 Non-functional requirements (from PRD, with measurement)

| ID | NFR | Target | How we measure it |
|---|---|---|---|
| NFR-1 | Read latency p99 | < 50 ms | gateway-side histogram, server-side histogram, alerted at 40ms p99 over 5m. |
| NFR-2 | End-to-end freshness p95 | < 2 s, event publish → projection visible to read API | `now() - event.occurred_at` at successful commit; histogram + alert at 1.5s p95 over 5m. |
| NFR-3 | Consumer availability | 99.95% monthly | uptime of "consumer lag < 30s" condition. |
| NFR-4 | Read API availability | 99.95% monthly | gateway success rate. |
| NFR-5 | Durability | no projection loss surviving zonal failure; full rebuildable from log | RPO=0 since source of truth is the log; RTO ≤ 1h for projection cold-start. |
| NFR-6 | Backfill throughput | full topic (~few million events) in ≤ 2 hours | wall-clock during scheduled drills. |
| NFR-7 | Idempotency | applying the same event N times yields the same projection state | invariant tested in suite, see §7.6. |
| NFR-8 | Correctness under reorder | applying events {A, B, C} in any order yields the same final state as applying them in sequence order | invariant tested. |

### 4.3 Out of scope (this milestone)

- Push to client (SSE/WebSocket).
- Multi-region active/active. We run primary in one GCP region with HA inside the region; DR is "rebuild from topic in secondary region", which is bounded by NFR-6.
- Per-customer order list projection.

---

## 5. High-level architecture

```
                    ┌────────────────────────────────────────────────┐
                    │              order service (Kotlin)            │
                    └──────────────────────────┬─────────────────────┘
                                               │ produces (Avro)
                                               ▼
                       orders.events.v1  (24 partitions, key=order_id)
                                               │
                                               ▼
              ┌────────────────────────────────────────────────────┐
              │     order-projections consumer (Kotlin / Spring)   │
              │  - one consumer group: order-projections           │
              │  - 1 worker thread PER assigned partition          │
              │  - per-(order_id) reducer + CAS write              │
              │  - DLQ on invalid transition / unparseable         │
              └────────────────┬──────────────────────┬────────────┘
                               │                      │
                               ▼                      ▼
                       Spanner table          orders.events.dlq.v1
                       order_projection
                               ▲
                               │  point reads
              ┌────────────────┴────────────────┐
              │ order-projections read API       │
              │  GET /orders/{id}/projection     │
              └──────────────┬───────────────────┘
                             │
                             ▼
                      Internal API gateway
                             │
                             ▼
                       customer app
```

Two deployables, **one codebase**, **one repo**:

- `order-projections-consumer` — Spring Boot app with Kafka listener, **no HTTP server**.
- `order-projections-api` — Spring Boot app with Spring Web, **no Kafka listener**.

Splitting into two deployables (same artifact, different profile/entrypoint) means we can scale and roll them independently. Read-side incidents must not block consumer progress and vice versa.

---

## 6. Consumer design — this is the load-bearing section

### 6.1 The model

We treat the projection as a **deterministic fold** over the set of events for an order:

```
projection(order_id) = reduce(events_for(order_id), initial_state, apply)
```

Two properties we require of `apply`:

- **Commutative-up-to-sequence**: given any permutation of events, the *final* state depends only on the highest sequence number that has been applied for each kind of state. Concretely: a `payment_captured` with sequence 7 cannot be overwritten by a `payment_authorized` with sequence 3 that arrives later.
- **Idempotent**: `apply(s, e) == apply(apply(s, e), e)`.

This is the only model that survives at-least-once + reordered delivery without bespoke per-event-type buffering.

### 6.2 Required event envelope (a hard ask of the producer)

Every event on `orders.events.v1` MUST carry, on the Avro envelope:

| Field | Type | Purpose |
|---|---|---|
| `event_id` | UUID | per-event identity for dedup |
| `order_id` | string | partition key |
| `sequence` | long (monotonic per `order_id`, gapless or gap-tolerant — see below) | ordering authority |
| `occurred_at` | timestamp-micros | wall clock for freshness metrics ONLY, never for ordering |
| `event_type` | enum | one of the 14 |
| `payload` | union of typed records | the body |
| `schema_version` | int | belt-and-braces alongside SR |
| `producer_id` | string | for debugging producer races |

**Critical:** `sequence` must be assigned by the order service **inside the same transaction that mutates the order**, e.g. `UPDATE orders SET version = version + 1 ...`. This makes `sequence` a logical clock. It does NOT have to be gapless across the topic — only monotonic per `order_id`. If a payment retry produces two events with the same `sequence`, that is a producer bug.

If the team confirms `sequence` is not currently emitted (likely), **this is a blocking dependency on the order service** and goes in §16 Open Questions / dependencies. The PRD does not mention it; we cannot ship NFR-8 without it. As a fallback, the order service can emit `aggregate_version` from its own row; semantically identical.

### 6.3 Reducer + CAS write

On every event:

1. **Decode + validate** envelope and payload against schema registry.
2. **Look up** current projection row by `order_id` (single-row read).
3. **Decide**:
   - If `event_id` is in `applied_event_ids` set → **drop** (duplicate). Increment `consumer.dedup.dropped`.
   - If `sequence <= projection.last_applied_sequence` → **drop** (stale / replayed). Increment `consumer.stale.dropped`.
   - Else → compute `next_state = apply(projection.state, event)`.
4. **Validate transition.** If the transition is illegal (e.g. `fulfilled` arriving when state is `null`/`created`), **two cases**:
   - The illegal transition is because we're missing an earlier event (gap): **park** the event in a small per-order buffer table for up to N=60s, then DLQ if the gap doesn't close. See §6.5.
   - The illegal transition is genuinely impossible (e.g. `refunded` before any payment): **DLQ immediately**, do not block consumer.
5. **Conditional write** to the store:

   ```
   UPDATE order_projection
      SET state = :next_state,
          last_applied_sequence = :sequence,
          last_applied_event_id = :event_id,
          applied_event_ids = applied_event_ids ∪ {event_id},
          updated_at = NOW()
    WHERE order_id = :order_id
      AND last_applied_sequence = :prev_sequence       -- CAS
   ```

   If the CAS fails, another worker beat us to it. We re-read and **restart at step 2** for this event. Bounded retry (3); if still failing, this is pathological — DLQ with reason `cas_storm` and alert.
6. **Commit the Kafka offset only after the write succeeds.**

This is correct under all of: duplicates, retries, out-of-order arrival within a partition (shouldn't happen but we don't rely on it), rebalance double-delivery, and consumer crash mid-write.

### 6.4 One worker per partition; no intra-partition parallelism

The single biggest source of "out of order" pain in projection consumers is async fan-out inside the consumer. We forbid it. Each Kafka partition is owned by exactly one thread (coroutine dispatcher in Kotlin). 24 partitions → up to 24 worker threads → ample for our event rate.

This trades throughput for correctness. At ~100 events/sec peak across 24 partitions (~4 events/sec/partition), per-partition serial processing is fine even with a 10ms write (400× headroom).

If we ever need more throughput, the safe scaling lever is **more partitions**, not more threads per partition.

### 6.5 Gap handling (the "park buffer")

If `sequence > last_applied_sequence + 1` we have a gap. Three options were considered:

1. **Apply anyway** — breaks correctness if `apply` is not commutative for that transition.
2. **Drop and DLQ** — too aggressive given producer/broker hiccups.
3. **Park briefly** — chosen.

The parker is a small in-memory map per worker `(order_id) → SortedSet<event>`. On each new event for an order with a gap, we attempt to drain in sequence order. Time-bound: 60s default (configurable). If the gap hasn't closed, DLQ the parked events with reason `sequence_gap_timeout` and continue from the next applicable sequence. We surface this as a metric — sustained `sequence_gap` is a producer bug we need to chase.

**Why in-memory and not in the store**: the park buffer is bounded (at peak: 100 ev/s × 60s = 6000 events worst case; nowhere near memory pressure), and rebuilding it on consumer restart is cheap because rebalancing will replay the relevant uncommitted offsets anyway.

### 6.6 DLQ topic

`orders.events.dlq.v1`, same partitioning. Payload = original Avro event + header keys:

- `dlq.reason` (`invalid_transition` / `sequence_gap_timeout` / `cas_storm` / `decode_error` / `unknown_event_type`)
- `dlq.original_topic`, `dlq.original_partition`, `dlq.original_offset`
- `dlq.error_message`
- `dlq.timestamp`

Ops can replay from DLQ via the same replay tool (§8).

### 6.7 Offsets and consumer config

- `enable.auto.commit=false`; we commit synchronously after successful store write.
- `isolation.level=read_committed` (defense against transactional producers if added later).
- `max.poll.records=50`, `max.poll.interval.ms=300_000`.
- `session.timeout.ms=45_000`, `heartbeat.interval.ms=15_000`.
- Static group membership (`group.instance.id` per pod) to reduce rebalance churn.
- Cooperative-sticky partition assignor.

---

## 7. Storage — the other load-bearing decision

### 7.1 Requirements on the store

- Point read by `order_id` at p99 < ~10ms (rest of the 50ms budget is gateway + serialization + network).
- Single-row CAS / conditional update.
- Atomic update of a row containing a moderately-sized JSON/struct (~3–10 KB per projection).
- Available zonally HA; backups; PITR ideally.
- On GCP, managed.
- Cost reasonable at ~1M rows, low-three-figures writes/sec peak, low-three-figures reads/sec peak.

### 7.2 Options considered

| Option | Verdict | Why |
|---|---|---|
| **DynamoDB** | **Rejected.** | We're on GCP. Cross-cloud network adds latency budget we don't have, breaks the single-cloud blast radius, doubles ops surface (IAM, billing, terraform providers), and there is no realistic "we'll move to AWS later" story justifying the friction. The PRD mentions it because someone is familiar with it; familiarity is not a sufficient reason. |
| **Cloud Spanner** | **Recommended primary.** | Strongly-consistent single-row reads ~5ms p99 at this scale, real CAS via row-level transactions, regional availability, schema we want, well-supported on GKE. Cost at our row count is the legitimate concern; see §7.5. |
| **Cloud SQL (Postgres 16)** | **Acceptable fallback** if Spanner cost is vetoed. | Single writer is a SPOF but at our write rate one HA pair is fine. `UPDATE ... WHERE last_applied_sequence = ?` gives us CAS. Slightly higher operational burden (failover, vacuum, connection pooling via PgBouncer). |
| **Firestore (native)** | Rejected. | Document model fits, but transaction semantics on hot keys are eventual under contention, and we have hot keys during flash sales (every event for the same order hits one document). Risk of CAS-storm under burst. |
| **Bigtable** | Rejected. | Excellent for high-volume KV but no native CAS on arbitrary columns (`checkAndMutate` exists but is single-cell and clunky for this shape), and the read latency / cost tradeoff isn't a win at 50K orders/day. Reconsider if volume grows 100×. |
| **Redis (Memorystore) as primary** | Rejected as primary. | Durability story is too weak for an ostensibly system-of-record-shaped read model that downstream services will start to depend on. Fine as a cache (see §7.4). |

### 7.3 Recommended primary: Cloud Spanner

Schema (one table):

```sql
CREATE TABLE order_projection (
  order_id              STRING(64)  NOT NULL,
  owner_customer_id     STRING(64)  NOT NULL,
  state                 JSON        NOT NULL,    -- the denormalized projection
  status                STRING(32)  NOT NULL,    -- denormalized for cheap indexed queries
  last_applied_sequence INT64       NOT NULL,
  last_applied_event_id STRING(64)  NOT NULL,
  applied_event_ids     ARRAY<STRING(64)>  NOT NULL,  -- bounded window, see below
  created_at            TIMESTAMP   NOT NULL,
  updated_at            TIMESTAMP   NOT NULL OPTIONS (allow_commit_timestamp=true),
) PRIMARY KEY (order_id);

CREATE INDEX order_projection_by_customer
  ON order_projection (owner_customer_id, updated_at DESC);  -- enables future per-customer projection
```

Notes on the schema:

- `state` is JSON so we can evolve the projection shape without DDL. The "shape" is owned by a versioned Kotlin sealed class (`OrderProjectionStateV1`), serialized via Jackson. Schema evolution is documented in §11.
- `applied_event_ids` is bounded — we only keep the last 100 event IDs; beyond that `last_applied_sequence` covers dedup. This keeps row size small.
- `owner_customer_id` is on the row so the read API can authorize without a join.
- A secondary index on `(owner_customer_id, updated_at DESC)` is the seed for the future per-customer projection (we are not building it, but we're making sure we don't paint ourselves into a corner — explicit PRD non-goal callout).

### 7.4 Optional Redis cache

Add **only if** load tests show NFR-1 is at risk. Memorystore (Redis), read-through, TTL 60s, invalidated by the consumer right after a successful Spanner write. At this scale I expect Spanner alone meets NFR-1; we will not preemptively add the cache.

### 7.5 Cost sanity check (Spanner)

- ~1M rows × ~5 KB ≈ 5 GB. Storage cost is rounding error.
- Provision **1 regional node** to start (well above what 100 writes/sec + 400 reads/sec requires). Cost ≈ ~$650–$750/mo USD list. Within the platform budget for a customer-facing latency-critical store.
- Stress-test will confirm we can drop to a smaller multi-node node-fraction (processing units) if Spanner pricing in 2026 supports it in this region.

If finance vetoes this number, switch to Cloud SQL Postgres HA (~$300/mo) — the rest of this design stays identical.

---

## 8. Replay & backfill

Two flavors:

### 8.1 Single-order replay (ops tool)

CLI / admin endpoint: `POST /admin/replay {order_id}`. Steps:

1. Read all events for `order_id` from Kafka. Since topic is keyed by `order_id`, this is a bounded partition scan — we use `KafkaConsumer.seekToBeginning` on the single partition `hash(order_id) % 24`, then filter. At ~few million topic events on 24 partitions, scanning one partition is a few hundred thousand events — seconds.
2. Sort events by `sequence`.
3. Tombstone the existing projection row (`DELETE` in a transaction).
4. Re-apply in sequence order via the **same reducer** the consumer uses. (Same code path; no "replay reducer" — that's how drift happens.)
5. Single transaction commit.

This is gated to ops (separate IAM role on the admin endpoint, separate gateway path, not exposed to the customer app gateway).

### 8.2 Full backfill

Triggered manually (initial seed + DR). Steps:

1. Spin up a parallel consumer group, e.g. `order-projections-backfill`, that reads from the **beginning** of the topic.
2. It writes to a **shadow table** `order_projection_shadow` with the same schema.
3. When caught up to the head of the topic (lag < a few seconds, sustained), we atomically swap by renaming the tables in a single Spanner DDL transaction, or — to avoid DDL drama — we cut over reads by config flag pointing the read API at the shadow table, then drop the old one.

Backfill throughput target: ≤ 2h for the full topic. With 24 partitions × ~4 events/sec/partition write headroom (the limit is the store, not the topic), we can replay millions of events well under 2h. Pre-prod drill required (§13).

### 8.3 Why not "just delete the row and let the consumer naturally re-apply"

Because the consumer is at the head of the topic — old events are no longer in the consumer's path. Replay must explicitly re-read from earlier offsets. Mixing replay into the live consumer would stall live processing for everyone.

---

## 9. Read API

### 9.1 Contract

`GET /orders/{order_id}/projection`

- Auth: gateway-enforced, injects `X-Customer-Id`.
- Authorization: server checks `projection.owner_customer_id == X-Customer-Id`. Else 404 (not 403 — don't leak existence).
- 200 response:

```json
{
  "order_id": "ord_01JX...",
  "status": "fulfilled",
  "state_version": 1,
  "last_applied_sequence": 17,
  "updated_at": "2026-09-12T14:08:22.123Z",
  "summary": {
    "line_items": [ ... ],
    "payment": { "authorized_amount": "...", "captured_amount": "...", "currency": "USD" },
    "fulfillment": { "shipments": [ ... ] },
    "refund": { "refunded_amount": "..." }
  }
}
```

- 404 if not found OR not owned by the caller.
- 503 if store is unavailable (the consumer being lagged is **not** a 5xx — we serve last known state and surface staleness via a header).

### 9.2 Staleness header

Add `X-Projection-Updated-At` and `X-Projection-Age-Ms`. Customer app can show "as of 3s ago" if it wants. Cheap, useful for debugging.

### 9.3 No write endpoints

The projection is read-only. Anyone tempted to add `POST /orders/{id}/projection` should be reminded the order service is the source of truth.

### 9.4 Caching headers

`Cache-Control: private, max-age=1`. Short, but enough to absorb a tab-refocus burst.

---

## 10. Capacity & performance

| Dimension | Steady-state | Peak (Black Friday) | Headroom design |
|---|---|---|---|
| Event ingest | 5 ev/s | 100 ev/s | Sized for **500 ev/s** (5× peak). |
| Read RPS | 80 RPS | 400 RPS | Sized for **2,000 RPS** (5× peak). |
| Projection rows | ~1M (1 yr) | grows ~18M/yr | Spanner row count is comfortable to 100M+. |
| Per-row size | ~3 KB | ~10 KB tail | We cap `state` blob at 64 KB and DLQ oversize. |
| Consumer pods | 4 (2× HA) | 8 | HPA on consumer lag, not CPU. |
| API pods | 3 (HA) | 8 | HPA on RPS. |

**Hot keys**: Even on a flash sale, an order is one row owned by one customer, so per-key hot spots are bounded. The hottest realistic case is a single order with rapid-fire state transitions (payment retries). Per-partition serial processing handles this exactly.

---

## 11. Schema evolution

- **Input events**: governed by Confluent SR with BACKWARDS compat. The consumer is built against the latest schema; old events deserialize cleanly because new fields are optional with defaults.
- **Projection state** (`OrderProjectionStateV1`): we version the *outer* type. When we need a breaking change to the projection shape, we introduce `V2`, run a backfill into a shadow column / table, then cut over. We never silently migrate readers.
- **API response**: additive only without version bump. Breaking changes require `/v2/orders/{id}/projection`.

---

## 12. Failure modes and how we handle them

| Failure | Detection | Behavior | Recovery |
|---|---|---|---|
| Consumer crash mid-event | Pod restart, Kafka lag alert | Uncommitted offset → event redelivered → dedup + CAS guarantee correctness | Automatic. |
| Spanner write fails (transient) | Write error counter | Retry with jitter (3 attempts), then push event back into a retry topic with delay; do not commit Kafka offset | Automatic via retry topic. |
| Spanner write fails (persistent, e.g. node loss) | Alert | Consumer halts (no offset advance), lag alert pages on-call | Spanner regional HA → SRE follows runbook. |
| Schema registry unreachable | Decode failures spike | Consumer keeps trying; the relevant events accumulate as Kafka lag, NOT loss | Page SRE; SR is a platform dependency. |
| Producer emits duplicate `sequence` (bug) | `consumer.sequence.collision` metric | One write wins via CAS, the other is dropped or DLQ'd | Page; this is a producer bug we have to chase upstream. |
| Producer skips a sequence | `consumer.sequence.gap` metric | Park buffer + 60s timeout, then DLQ | Investigate producer. |
| Genuinely illegal transition | `consumer.transition.invalid` metric | DLQ | Operator decides: replay (if bug) or accept (if data). |
| Store hot row (single order, very rapid events) | CAS retry count metric | Per-partition serial processing means no CAS storm; if seen, investigate | n/a expected. |
| Backfill blows up cost | Spanner CPU alert | Throttle backfill consumer parallelism; it's a separate consumer group | Tunable. |
| Region outage | GCP status + our SLO burn | Read API degrades to 503; rebuild in DR region within RTO=1h | Documented runbook. |

---

## 13. Testing strategy

- **Reducer unit tests**: golden-file driven. For each event type, exhaustively test idempotency (`apply` twice = once) and order-insensitivity for the commutative cases.
- **Property-based tests** (jqwik or Kotest property): for any random permutation of a generated valid event sequence, the final state is identical. This is how we sleep at night on the reorder concern.
- **Contract tests** against Schema Registry: every supported schema version produces a parseable Avro payload our reducer accepts.
- **Integration tests** with Testcontainers: Kafka + Spanner emulator. Real Kafka rebalance scenarios using two consumer instances.
- **Replay drill in pre-prod, monthly**: full backfill from topic start; measure wall-clock against NFR-6.
- **Load test in pre-prod**: 2,000 RPS reads + 500 ev/s writes, sustained 30 min, then 10× burst for 60s. Pass = NFR-1 and NFR-2 hold.
- **Chaos**: kill consumer pods during a synthetic event storm; assert no projection drift after recovery (diff against shadow rebuild).

---

## 14. Observability

### 14.1 Metrics (Prometheus, scraped by our existing stack)

- `consumer.events.consumed{event_type, outcome}` — counter.
- `consumer.events.duplicate.dropped`, `consumer.events.stale.dropped`.
- `consumer.events.dlq{reason}`.
- `consumer.lag.records{partition}` — from Kafka admin.
- `consumer.lag.seconds` — `now - event.occurred_at` at commit; **this is the NFR-2 signal**.
- `consumer.write.duration` — histogram.
- `consumer.cas.retries` — histogram.
- `consumer.park.depth{partition}`, `consumer.park.evicted_to_dlq`.
- `api.read.duration` — histogram, **this is NFR-1**.
- `api.read.status{code}`.
- `store.spanner.cpu` (from GCP exporter).

### 14.2 Tracing

OpenTelemetry, propagated from gateway → API → Spanner. On the consumer side, each event gets a trace with span attributes `order_id`, `event_id`, `sequence`, `partition`, `offset`. Linked from producer trace if the producer team propagates context (separate ask).

### 14.3 Logs

Structured JSON, INFO at lifecycle events, WARN at retries/parks, ERROR at DLQ/CAS-storm. PII-sensitive fields (customer email, address) are redacted at log time by a Jackson serializer; we log `order_id` and `customer_id` but never raw addresses.

### 14.4 SLOs

| SLO | Window | Burn alert |
|---|---|---|
| Read availability ≥ 99.95% | 28d | 2% burn over 1h pages |
| Read p99 ≤ 50ms | 28d | 2% burn over 1h tickets, 5% pages |
| Freshness p95 ≤ 2s | 28d | 2% burn pages |

### 14.5 Dashboards

One dashboard per concern: ingestion, freshness, read latency, store health, DLQ. Shared with customer-app team for cross-debugging.

---

## 15. Rollout plan

16-week plan to GA on Sept 30. (We'll need to compress if the producer-side `sequence` work slips — see §16.)

| Week | Milestone |
|---|---|
| 1 | This TDD signed off. Producer-side `sequence` field agreed with order-service team. Spanner instance provisioned in pre-prod. |
| 2–3 | Reducer + idempotency + property tests. No Kafka, no store yet. |
| 4–5 | Consumer skeleton against test Kafka; integration with Spanner emulator. |
| 6 | DLQ + park buffer. Backfill mode (shadow table). |
| 7 | Read API + gateway integration in pre-prod. |
| 8 | Load test + freshness test in pre-prod. Adjust pod sizing. |
| 9 | Chaos drills. Run full backfill drill. |
| 10 | Shadow-write to prod (consumer writes to a `_shadow` table; nothing reads it). Validate no drift vs. order service for a week. |
| 11 | Read API behind feature flag for 1% of customer app traffic. |
| 12 | Ramp to 10%, 50%, 100% with kill switch back to polling. |
| 13 | Cut customer-app off polling order service entirely. |
| 14 | Burn-in. Tune alerts. |
| 15 | Buffer / catch-up. |
| 16 | Holiday freeze begins. |

### 15.1 Rollback

- Read API: feature flag flips customer app back to polling order service. Order service polling endpoint is **not** retired until after holiday season.
- Consumer: stop pods. Topic retains events. Restart later from committed offsets.
- Storage: shadow-table approach means we always have a known-good table to fall back to during cutover.

---

## 16. Open questions and dependencies

1. **`sequence` field on events.** Does the order service emit a monotonic per-`order_id` sequence today? If not, who owns adding it, and what's their ETA? **Blocking.** Without it we can offer best-effort correctness only.
2. **Producer idempotency.** Is the order service's Kafka producer configured with `enable.idempotence=true` and `acks=all`? If not, duplicate events in-partition are likely. Cheap fix, big payoff.
3. **What does "out of order" mean concretely?** Logs / examples from the team that observed it. Without specifics we may be designing for the wrong failure mode. Ask SRE for the post-mortems.
4. **Auth model.** Confirm the gateway always injects `X-Customer-Id`. Confirm CS/admin tools route through a different gateway path that bypasses the customer-id check.
5. **PII / data residency.** Is order data in scope for any regional residency requirement (EU customers in particular)? If so, we may need a per-region Spanner instance and a regional consumer group. Currently assumed single region.
6. **Retention.** PRD doesn't say. We propose: projection rows retained indefinitely; DLQ retained 30 days; consumer offsets standard.
7. **Cost ceiling.** Confirm finance accepts Spanner cost (§7.5). If not, switch to Cloud SQL Postgres — design works as-is.
8. **Per-customer projection.** PRD calls it out as future. Do we want to lay the indexed groundwork (§7.3) now or skip it? Recommendation: include the index, it's cheap.
9. **Replay tool surface.** CLI? Admin HTTP? Both? Who's authorized?
10. **SLA vs. SLO with customer-app team.** PRD gives engineering targets; do we need a contractual SLA exposed to customers ("order status updates within 5 seconds")?
11. **Schema Registry compatibility direction.** PRD says BACKWARDS. Confirm with platform team that no producer is on a non-compatible-mode topic; if any are, the consumer needs defensive handling.

---

## 17. Decision log (for the record)

| Decision | Choice | Why we did it this way |
|---|---|---|
| Cloud | GCP only | Platform constraint; cross-cloud not justified. |
| Primary store | Cloud Spanner | Strong consistency + CAS + managed HA on GCP at acceptable cost. |
| Fallback store | Cloud SQL Postgres | Identical design with a single-writer HA pair if Spanner is rejected. |
| Cache | None initially | Spanner meets NFR-1 at our load; add only if load test says otherwise. |
| Consumer language | Kotlin / Spring Boot | Team skill; matches order service. |
| Ordering authority | Per-`order_id` `sequence` from producer | Arrival order is not trustworthy under at-least-once. |
| Idempotency | `event_id` dedup + `last_applied_sequence` CAS | Two independent guards. |
| Parallelism | One worker per partition, serial | Removes the #1 source of reorder bugs. |
| Replay | Same reducer, shadow-table swap | One code path; no drift. |
| DLQ | Yes, with reason taxonomy | Required by PRD. |
| Deployables | Two (consumer, API) from one repo | Independent scaling and blast radius. |
