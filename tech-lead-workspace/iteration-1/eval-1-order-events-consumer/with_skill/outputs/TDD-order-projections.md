# TDD: Order Projections Service

**Author:** Tech Lead (on behalf of Platform team)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-order-events-consumer.md`
**Workload category:** Event-driven / streaming consumer (Kafka)

---

## 1. Problem & context

The customer-facing app currently polls the order service every 5 seconds for
status (~120 RPS of pure waste, and up to 5s of visible lag). We want sub-second
order status without that polling cost. Order state already lives as a stream
of domain events on Kafka `orders.events.v1` (~14 event types, partitioned by
`order_id`, 24 partitions). The right move is a **read-model projection**: a
long-running consumer maintains a denormalized per-order document in a
low-latency store, and the customer app reads from a thin HTTP endpoint behind
the existing API gateway.

Driver: holiday ramp. Black Friday last year hit 18K orders/day; the polling
load and lag will get worse as traffic grows, and the customer-app team has
this on their critical path. Hard deadline: Sept 30 (Q3).

Consumers of this projection: the customer-facing web/mobile app (via the
existing API gateway). The order service stays the source of truth; we are
strictly a derived read-only view.

## 2. Scope

**In scope**

- Kafka consumer service that subscribes to `orders.events.v1` and maintains a
  per-`order_id` projection document.
- Read API: `GET /orders/{order_id}/projection` behind the existing gateway.
- Per-event state machine (applies the ~14 event types, rejects illegal
  transitions, DLQs them).
- DLQ topic + alarms + an ops replay tool for a single `order_id` (re-read all
  events for that key from the start of retention and rebuild).
- Full back-fill path (cold start / disaster rebuild from topic-start).
- Schema-registry integration (Avro, BACKWARDS compat).
- Observability: lag per partition, DLQ rate, projection write latency, read
  API SLOs.

**Out of scope**

- Replacing the order service as source of truth.
- Cross-order aggregations (e.g., "customer's last 10 orders"). Called out as a
  **future projection** — design must not paint itself into a corner; see
  Extensibility.
- Historical analytics — separate batch pipeline owns that.
- Write APIs of any kind. This service is read-only to clients.
- Authn/authz logic beyond what the gateway already enforces.

## 3. High-level approach

One Kotlin service (consistent with the order service language and the team's
Kotlin/Java split), deployed on GKE as a single Deployment with N replicas
where `N <= 24` (partition count). It runs two concerns in one binary,
clearly separated:

1. **Consumer** — Kafka consumer group `order-projections-v1`, manual offset
   commit *after* the projection write succeeds. Per-partition serial
   processing (preserves order within an `order_id`). On apply, it writes
   the new projection version with a conditional write keyed by
   `(order_id, projection_version)` (see §4b — this is how we beat
   out-of-order events and duplicates in one mechanism).
2. **Read API** — a small HTTP server (Spring Boot WebFlux or Ktor) that
   serves `GET /orders/{order_id}/projection` from the same store. No
   business logic in the read path — single point-read.

Storage: **DynamoDB Global Tables is NOT a great fit on GCP**. The PRD says
GCP/GKE and the team is leaning DynamoDB; that lean needs to be challenged.
Recommendation: **Spanner** (or Postgres on Cloud SQL with a single-row JSON
document per order) as the primary, with a tight argument in §5 for why
**not** DynamoDB on GCP. If the team has already committed to a cross-cloud
story and AWS is an option, DynamoDB becomes viable — surfaced as Open Q1.

Diagram (prose):

```
[order-service] --produces--> [Kafka: orders.events.v1, 24 parts, Avro/SR]
                                       |
                                       v
                          [order-projections consumers (Kotlin, GKE)]
                                       |
                       reject -> [DLQ: orders.events.v1.dlq]
                                       |
                       apply  -> [Spanner: order_projections]
                                       ^
                                       |  point read
                          [Read API: GET /orders/{id}/projection]
                                       ^
                                       |
                              [API Gateway] <-- [customer app]
```

Boxes: order-service producer, Kafka topic with schema registry, the
projections consumer pod set, the DLQ topic, the Spanner table, the read API,
the existing gateway. Arrows: produce → consume → apply (conditional write)
→ point-read. Single error arrow from consumer → DLQ.

## 4. Detailed design

### 4a. Broker & topic model

- **Broker:** Confluent Kafka (Pub/Sub-mirrored — i.e., we treat it as Kafka,
  not as Pub/Sub). Already in place. Do not change brokers; the PRD is firm.
- **Topic consumed:** `orders.events.v1`. 24 partitions, partitioned by
  `order_id`. Retention: assume 7 days (confirm — Open Q2). At ~50K
  orders/day with mean ~5 events per order = ~250K msgs/day = ~3 msgs/sec
  steady-state. Black Friday peak ~5K events/min worst case ≈ 80 msgs/sec.
  Tiny by Kafka standards; the design is bottlenecked on storage, not the
  consumer loop.
- **Schema:** Avro on Confluent Schema Registry, `BACKWARDS` compatibility.
  Producer is the order service.
- **Consumer group:** `order-projections-v1`. Single consumer group. This is
  a read-model fan-out destination, not a competing consumer.
- **DLQ topic:** `orders.events.v1.dlq` (we create and own this). Same
  partition count (24) and Avro envelope wrapping the original payload +
  failure metadata (see §4d).

### 4b. Delivery semantics, ordering, and the out-of-order problem

This is the section the team most needs us to get right. The PRD specifically
flagged "we keep getting bitten by out-of-order events" and "duplicates from
at-least-once".

**What Kafka actually guarantees on this topic:**

- **Within one partition**, messages are strictly ordered.
- **Across partitions**, no ordering. Since partitioning is by `order_id`,
  all events for a single order land on **one** partition and are therefore
  ordered — *as long as* the producer publishes them in order and the broker
  doesn't reshuffle.
- **At-least-once delivery.** Duplicates happen on producer retry, on
  consumer rebalance, and on offset-commit-after-crash races.

**Where "out-of-order" actually comes from on this stream** (named explicitly
so we don't design around the wrong cause):

1. **Producer-side reordering during retry.** The order service retries a
   publish that timed out; an earlier message that succeeded but the ack was
   lost is followed in the log by what should be the "next" message. Kafka
   producer with `enable.idempotence=true` + `max.in.flight.requests.per.
   connection<=5` + `acks=all` prevents this. **Action:** verify the order
   service producer config — if not set, fix it upstream. (Open Q3.)
2. **Cross-partition reordering during rebalances** — the PRD mentions this.
   This should not happen for events of a single `order_id` because they
   share a partition; if it *is* happening, it means either (a) the producer
   doesn't always partition by `order_id` (some events emitted by a
   different service with a different key), or (b) someone has changed
   partition count and old vs new hashes route the same key differently.
   Both are upstream bugs we need to confirm and close. (Open Q4.)
3. **Genuine application-level out-of-order** — e.g., `payment_captured`
   processed before `payment_authorized` because the upstream service emits
   them from two different DB transactions racing. This is real and we must
   defend against it in the consumer regardless of broker behavior.

**Our defense — `event_version` + conditional write, NOT timestamps.**

Every event MUST carry a monotonic per-`order_id` `event_version` (integer,
starts at 1, +1 per event for that order). If the producer doesn't already
emit one, **that's the blocking design change** — without a monotonic
per-order sequence number, no read-model projection over an at-least-once
unordered stream is correct. (Open Q5 — this is design-flipping.)

Given `event_version`, the apply step is:

```
read current projection P for order_id  (or empty)
if event.version <= P.applied_version: drop (duplicate or stale reorder)
if event.version == P.applied_version + 1:
    new_P = transition(P, event)
    conditional write where applied_version == P.applied_version
    on conflict (someone else won the race): re-read, retry once
if event.version > P.applied_version + 1:
    # gap — out-of-order arrival
    park event in a per-order "pending" buffer (in Spanner, same row, as a
    JSON list capped at N=50 entries, TTL 10 min)
    do NOT commit Kafka offset for this message yet — wait for the gap to
    close
    if gap doesn't close within 60s, DLQ the gap-blocking event with
    reason=gap_not_closed
```

This gives us **effectively-once** semantics on the projection:
- Duplicates are dropped by `event.version <= applied_version`.
- Out-of-order within a partition is reconciled by the pending buffer.
- Out-of-order across partitions is impossible if partition-by-`order_id`
  holds. If it doesn't hold (Open Q4), the pending buffer covers a bounded
  window of reordering across partitions too.

**End-to-end guarantee provided to consumers of the read model:** the
projection reflects events in `event_version` order for each `order_id`, with
duplicates suppressed. We do **not** promise the projection is up to date —
read API responses include `applied_version` and `event_time` so callers can
detect staleness.

### 4c. Offset / checkpoint management

- **Auto-commit off.** Manual commit after successful apply (or after DLQ).
- **Commit cadence:** per message for now. At 80 msgs/sec peak, the offset
  commit RPC overhead is irrelevant. If we ever need to batch, batch within
  a partition only; never across partitions in the same commit.
- **Apply ordering:** within a partition, strictly serial. Across partitions,
  parallel — one worker thread per assigned partition.
- **Crash recovery:** offsets are committed only after the conditional write
  succeeds; on crash, replay re-applies the last in-flight message, and the
  `event_version <= applied_version` check makes it a no-op. This is what
  makes the system safe under at-least-once.
- **Gap-pending state lives in Spanner**, NOT in memory. A pod restart must
  not lose pending events. (This is also why we don't commit the Kafka
  offset for the gap-blocking message until the gap closes — but we **do**
  commit offsets for events that landed in the pending buffer, because they
  are now durably stored in Spanner.)

### 4d. Poison-message / DLQ handling

A message is DLQ'd, never silently dropped, in these cases:

| Trigger | DLQ reason field |
|---|---|
| Avro deserialization fails (schema mismatch the consumer cannot handle) | `schema_incompatible` |
| Illegal state transition (e.g., `fulfilled` on a never-created order) | `illegal_transition` |
| `event_version` is malformed or missing | `missing_version` |
| Gap pending buffer full or 60s gap timeout | `gap_not_closed` |
| Application exception on apply, after 3 retries with backoff | `apply_failed` |
| Spanner write returns non-retryable error (constraint violation only — most Spanner errors are retryable) | `store_write_failed` |

DLQ envelope (Avro):

```
{
  original_payload: bytes,
  original_topic, original_partition, original_offset,
  failure_reason: enum,
  failure_message: string (sanitized — no PII),
  consumer_attempt: int,
  ts_failed: timestamp,
  consumer_version: string
}
```

**Replay tool** (`ops/replay-order.sh order_id`):
- Reads all messages for that `order_id` across the entire topic retention
  window (scan with a partition-level read of just that key's partition,
  filter client-side).
- Wipes the existing projection row.
- Re-applies in `event_version` order.
- Writes a structured audit record.

**Alerting** on DLQ rate:
- > 5 messages/hour to DLQ for any reason → ticket.
- > 50 messages/hour OR any single reason spiking → page.
- Specifically, `gap_not_closed` rate > 0.1% of input rate → page (means
  out-of-order is no longer a tail event).

### 4e. Concurrency & parallelism

- **Pods:** start with 4 pods, each consuming 6 partitions. Headroom to scale
  to 24 pods × 1 partition = 24× the per-pod throughput before we run out of
  partition headroom. At 80 msgs/sec peak, this is comically over-provisioned.
- **Per pod:** one worker thread *per assigned partition*. Serial within a
  partition (mandatory for ordering). 6 threads of work per pod is trivial.
- **Per-message timeout:** 5s wall-clock budget. The bound on apply is
  basically one Spanner read + one Spanner conditional write — sub-50ms p99.
  A 5s budget is paranoid.
- **Kafka session timeout / max poll interval:** session 10s, max poll
  interval 5min. We are nowhere near these.

### 4f. Back-pressure

- The slowest downstream is Spanner. Spanner can do tens of thousands of
  small conditional writes per second on a single 1-node instance.
- If Spanner gets slow (regional issue), consumer lag grows. Kafka retention
  is the real ceiling: at 7-day retention and 250K msgs/day, we have 1.75M
  messages of buffer. We will hit lag alarms long before retention loss.
- **Lag alarms:** see §6 (Performance / Observability).
- No upstream rate-limiting needed — the order service produces what it
  produces; we either keep up or we lag.

### 4g. Stateful processing

- All "state" is in Spanner: the projection row itself + the pending-events
  buffer on the same row. We deliberately **do not** keep state in process
  memory — pod restarts must be no-ops semantically.
- The only in-memory state is the active Kafka offset per partition (managed
  by the Kafka client) and per-partition work queues (bounded, size 100).

### 4h. Schema evolution

- Schema Registry mode: `BACKWARDS` (already enforced).
- Adding a new optional field is always safe. Removing a field needs a
  forward migration: the consumer must handle the field's absence first,
  then it can be removed from the schema.
- Adding a **new event type**: the consumer's state machine has a default
  branch that DLQs unknown types with reason `unknown_event_type`. The
  alarm on that DLQ reason is what tells us we forgot to update the
  consumer. (Today's ~14 types will grow; this is the variation point.)
- Breaking change → new topic `orders.events.v2`. We stand up a parallel
  consumer subscribed to v2 writing to the same Spanner table; v1 consumer
  is drained after producers cut over.

### 4i. Data model — Spanner

Table `order_projection`:

```
order_id            STRING(64)   NOT NULL  PRIMARY KEY
applied_version     INT64        NOT NULL
projection_json     JSON         NOT NULL    -- the denormalized document
pending_events_json JSON                     -- bounded buffer, see §4b
last_event_time     TIMESTAMP    NOT NULL
last_updated        TIMESTAMP    NOT NULL    -- commit_timestamp()
schema_version      INT64        NOT NULL    -- of the projection_json shape
```

Why one big JSON column for the projection: the read API serves it as-is,
and we never query inside it. Future cross-order projections (per-customer
list) are a **separate** table with their own partition key — this is what
the PRD's "called-out future projection" requires us to not block.

Conditional write is the standard Spanner read-modify-write transaction
that asserts `applied_version` hasn't changed.

### 4j. Read API

- `GET /orders/{order_id}/projection`
- Single Spanner point read by primary key. p99 < 10ms expected; p99 < 50ms
  SLO has comfortable headroom.
- Response includes `applied_version`, `last_event_time`, and the projection
  body. Caller can detect staleness.
- 404 if no row exists yet. (Possible race: order just created, projection
  hasn't applied. Customer app should treat 404 in the first few seconds
  after order creation as "still propagating" and retry.)
- Auth: rely on the existing API gateway. Inside this service we accept the
  authenticated subject from the gateway header and check `subject can read
  order_id` via the existing authz library (same as order service uses).

### 4k. External dependencies — failure behavior

| Dependency | If down | Effect |
|---|---|---|
| Kafka | Consumer cannot poll | Lag grows; reads still serve stale data. Page after 5min of no polls. |
| Schema Registry | New schema versions unresolvable | Cache last-known schemas in memory; serve from cache. If a never-seen schema arrives, DLQ with reason `schema_unavailable`. |
| Spanner | Reads and writes fail | Read API returns 503 with `Retry-After`. Consumer pauses (Kafka offsets not committed) and lag grows. |
| API Gateway | Read API unreachable from clients | Customer app falls back to polling order service. Document this fallback in the customer-app runbook. |

## 5. Alternatives considered

### Storage — the headline alternative

We were asked to take a principal view on storage. The PRD team is leaning
DynamoDB. Here is the honest tradeoff.

**Option A: Spanner (recommended on GCP).**
- *Pros:* Strongly consistent on a primary-key conditional write — perfect
  for our `applied_version` CAS. Single-region p99 < 10ms point reads at
  this volume. Schema is real (we get `applied_version INT64 NOT NULL` as a
  constraint, not a convention). Native on GCP/GKE. Multi-region is a
  config flip when it's needed.
- *Cons:* Most expensive of the three. Minimum 1-node = ~$650/mo per region
  even at zero load. Overkill for 80 msgs/sec.
- *Verdict:* **Recommended** despite cost because (a) we are explicitly
  solving an out-of-order/dedupe problem where a single-key CAS is the
  cleanest primitive, and (b) operational simplicity (no sharding, no hot
  partition story) wins against ~$650/mo at our scale.

**Option B: Postgres on Cloud SQL.**
- *Pros:* Cheapest. Team knows it. `UPDATE ... WHERE applied_version = ?`
  is the same CAS pattern, even simpler. JSONB column for the document.
- *Cons:* Single-writer architecture; HA failover is real downtime
  (~30–60s). Read replicas are async — using them for the read API
  reintroduces a different staleness story than the one the projection
  itself provides. MVCC bloat on a heavily-updated narrow table is a real
  ops burden over years.
- *Verdict:* **Viable second choice.** If the team is cost-sensitive and
  Spanner's price is unjustifiable for an internal-ish read model, take
  this. It is *not* a worse architecture, just a busier ops story.

**Option C: DynamoDB (the team's lean).**
- *Pros:* Conditional writes (`ConditionExpression: applied_version = :v`)
  are first-class — semantically identical to our Spanner CAS. Pay per
  request, basically free at our volume. Single-digit-ms p99 reads.
- *Cons (and this is the principal take):*
  - **We are on GCP.** Running the projections store on AWS means
    cross-cloud egress on every consumer write and every read API call.
    At 80 msgs/sec + customer app reads (which replace 120 RPS of polling
    — probably 30–60 RPS of actual reads), egress is small but **latency
    is not**: GCP↔AWS round-trip is 20–80ms depending on region pair,
    which blows our p99 < 50ms read SLO unless we colocate (deploy the
    read API on AWS too — but then the auth gateway is on GCP, and we've
    just sprawled the architecture).
  - **Operational surface.** New AWS account, new IAM model, new
    monitoring story, new on-call escalation. For one table.
  - **It only wins if we are already multi-cloud or planning to be.** If
    that's a roadmap decision the team has actually made, DynamoDB is
    fine. If it isn't, this is the wrong reason to start.
- *Verdict:* **Not recommended on GCP-only.** The DynamoDB conditional-
  write story is great, but Spanner gives us the same semantics in-cloud.

**Option D: Redis (also floated).**
- *Pros:* Fastest read latency. Cheap.
- *Cons:* Not durable in the way we need. Conditional writes via WATCH/
  MULTI/EXEC are doable but tricky at scale. AOF/RDB durability gaps
  during failover can lose the most recent applies — meaning we lose the
  `applied_version` and the next event's CAS races confusingly. Wrong
  primitive for an authoritative read model. Could be added later as a
  cache in front of Spanner if read latency becomes a problem (it won't).
- *Verdict:* Rejected as primary store. Possible later cache.

### Other alternatives

- **Kafka Streams / ksqlDB with RocksDB-backed state store.** Tempting —
  the projection *is* a stateful streaming computation. Rejected because:
  (a) Kafka Streams requires JVM consumer topology, more complex to
  operate, RocksDB state recovery on rebalance can be slow (minutes); (b)
  the read API would have to query a Kafka Streams interactive-query
  endpoint, which means each pod serves a slice of keys and the read API
  has to route — significant added complexity for a problem we don't have
  (our read volume is tiny).
- **Skip the projection, scale the order service for direct reads.** Costs
  more long-term (every read is a join across orders/payments/fulfillment
  tables), couples the customer app to the order service's availability,
  and doesn't solve the polling problem. Rejected.
- **Do nothing — keep polling.** 120 RPS today, growing; ships 5s of lag.
  Rejected: the customer-app team has this as a Q3 dependency.

## 6. Non-functional requirements

### Reliability

- **Message loss budget:** zero (we have a durable broker and durable store;
  the design has no place a message can be silently dropped).
- **DLQ rate budget:** < 0.01% of input. Verified by alarm on DLQ rate.
- **Mechanism:** offset commit after apply; `event_version` dedupe; DLQ for
  all non-success paths.

### Performance

- **Read API:** p50 < 10ms, p95 < 30ms, p99 < 50ms (PRD). Verified by load
  test against staging Spanner with realistic projection sizes.
- **End-to-end (event publish → projection updated):** p95 < 2s (PRD), p99
  < 5s. Verified by synthetic probe: emit a test event every 30s with a
  client-generated UUID, watch for it to land in the projection.
- **Consumer lag SLO:** p95 lag < 30s. p99 < 2min. Alert at p95 > 60s for
  5min, page at p95 > 5min.
- **Throughput headroom:** designed for 80 msgs/sec; tested to 1000
  msgs/sec; partition count gives us 24× concurrency headroom.

### Observability

- **Metrics (Prometheus / GCP Monitoring):**
  - `consumer_lag{partition}` — primary health signal.
  - `events_processed_total{event_type, outcome}` where outcome ∈
    {applied, deduped, pending_buffered, dlq}.
  - `apply_duration_seconds{event_type}` — histogram.
  - `dlq_total{reason}` — histogram, alarm dimensions.
  - `pending_buffer_size{order_id_hash_bucket}` — should be near zero;
    spike means out-of-order is happening.
  - `read_api_duration_seconds{status}` — RED.
  - `spanner_write_conflicts_total` — CAS retries; spike means a hot key
    (one order receiving many simultaneous events from the producer).
- **Logs:** structured JSON, including `order_id`, `event_id`,
  `event_version`, `applied_version`, `partition`, `offset`. No PII in
  logs (order line items will have addresses — strip).
- **Traces:** OpenTelemetry. Trace from consumer-poll through apply
  through Spanner write. Sampled at 1% normally; 100% on errors.
- **Alerts:**
  - Lag p95 > 60s for 5min → page.
  - DLQ rate > 50/hour → page.
  - DLQ rate > 5/hour OR any new `reason` value → ticket.
  - `pending_buffer_size > 10` for any order for > 2min → ticket
    (out-of-order is starting to bite).
  - Read API 5xx > 1% for 5min → page.
- **Runbook:** placeholder, must be written before launch:
  `<link to runbook — to be added>`. Must cover: DLQ replay, single-order
  rebuild, consumer-stuck-on-poison-message, Spanner regional outage.

### Resilience

- All external calls have explicit timeouts (Spanner 1s read, 2s write;
  Schema Registry 500ms, cached).
- Exponential backoff with jitter on Spanner retry and Kafka publish retry
  to DLQ.
- Pod-restart safe (state is in Spanner; in-flight messages are re-read).
- Bulkhead: one stuck partition does not block others (per-partition
  worker thread).

### Scalability

- Capacity: 80 msgs/sec peak today, growth horizon ~5× over 2 years
  (assumption — Open Q6 for growth target).
- Scaling axis: horizontal, up to 24 pods (partition count). Past 24 we'd
  need to bump partition count, which is a careful operation (consumer
  must drain and restart on the new partition map). Documented but not
  needed for years.
- No hot-key risk: one `order_id` receives at most a few events/sec
  during heavy activity (auth, capture, ship), and they're serialized on
  one partition.

### Security

- **AuthN:** API gateway already does this. We accept gateway-issued
  identity headers and refuse direct calls (firewall the pods to the
  gateway only).
- **AuthZ:** subject must own the `order_id` or have a service role.
  Reuse the existing authz library used by the order service.
- **Data classification:** order projections contain PII (shipping
  address, possibly name/email in line items). Mechanism: Spanner
  encryption at rest (default), TLS in transit, structured logs strip
  PII, traces redact attribute values for fields marked PII in the
  Avro schema.
- **Secrets:** Spanner credentials via GKE Workload Identity. Kafka SASL
  credentials from Secret Manager.
- **Supply-chain:** built artifacts signed; base image scanned in CI;
  dependency lockfile committed.

### Cost

- Estimated: ~$650/mo Spanner (1 node) + ~$200/mo GKE pods + Kafka
  cost is shared infra. Order of magnitude: < $1500/mo.
- Cost ceiling alarm: $2500/mo.
- Biggest driver is Spanner; if cost is a blocker, the Postgres
  alternative cuts it ~3×.

### Compliance & privacy

- GDPR: order PII is replicated here. We honor delete-from-source by
  consuming a `customer_deleted` or `order_redacted` event (Open Q7 —
  does this event exist? if not, the order service team needs to add it).
- Retention: projection rows live as long as the source order does. We
  don't independently expire.

### Disaster recovery

- RPO: 0 for the projection (it's derivable from the event log) provided
  Kafka retention covers the recovery window. RPO for Kafka is whatever
  the Kafka cluster's RPO is — not our problem here.
- RTO: < 1 hour for a regional Spanner outage if we have a multi-region
  config; ~4 hours for a full rebuild from topic-start at 80 msgs/sec
  × the time to scan all retained events (~250K × 7 days = 1.75M events,
  ~5 hours single-threaded, ~15 minutes parallelized across 24
  partitions). Documented; not currently a blocker.
- Multi-region: not required for v1 (the order service itself isn't
  multi-region). Spanner makes it a config change if we need it later.

### Maintainability

- Tests: unit (state machine transitions exhaustively), integration
  (Testcontainers Kafka + Spanner emulator), contract (Avro schema
  pinned against the order service's published schema), load (k6 or
  Gatling at 1000 msgs/sec for 30min).
- Ownership: platform team (the team building this).

### Omitted explicitly

- **Multi-region active-active:** out of scope for v1; the source-of-truth
  order service is single-region.
- **Real-time analytics:** explicitly not this service.
- **Read-side caching:** not needed at the target read volume; Spanner is
  fast enough.

## 7. Risks & failure modes

| Risk | Likelihood | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Producer doesn't emit `event_version` | **High** (unknown today) | **Whole feature** | Block the project until confirmed — see Open Q5. | N/A — design-flipping. |
| Producer occasionally publishes events for the same order on different partitions (partition-key bug) | Medium | Per-order ordering breaks | Add monitoring on `pending_buffer_size`; investigate producer; pending buffer covers a 60s reorder window. | Producer fix + DLQ replay for affected orders. |
| Consumer falls behind during a flash sale | Medium | Customer app sees stale state | Lag alarm fires; horizontal scale up to 24 pods; back-pressure is graceful (reads still serve stale data, not errors). | Scale up; lag recovers in minutes. |
| Spanner regional outage | Low | Reads fail, consumer stalls | Read API returns 503; consumer pauses; customer app falls back to polling order service. | Spanner recovers; offsets advance from where they stalled (no data loss). |
| Schema registry change breaks consumer | Low (BACKWARDS compat enforced) | Consumer DLQs all new messages | DLQ alarm fires within minutes. | Roll back consumer or schema; replay DLQ. |
| Poison message stops a partition | Low | One order's projection stale, but other partitions fine | Per-partition isolation; DLQ after 3 retries. | Inspect DLQ; fix bug; replay. |
| `applied_version` CAS thrash on a very hot order | Low at our scale | One order's apply slows | Single-thread per partition serializes events for an order; conflicts are rare. | Monitor `spanner_write_conflicts_total`. |
| Pending buffer fills (out-of-order persistent) | Low if upstream is healthy | DLQ rate climbs | Alarm on `pending_buffer_size`. | Diagnose upstream; replay DLQ once upstream fixed. |

## 8. Rollout & migration

1. **Phase 1 (week 1–2):** stand up service in staging. Consume from
   staging Kafka. No client traffic. Verify lag, DLQ rate, projection
   correctness via shadow comparison: for sampled orders, fetch via order
   service and via projection, diff, alarm on divergence.
2. **Phase 2 (week 3):** production deploy with consumer running but
   read API not yet wired to gateway. Shadow comparison in production
   for 1 week.
3. **Phase 3 (week 4):** customer app rolled forward with a feature
   flag, 1% → 10% → 50% → 100% over a week. Flag default: off.
4. **Phase 4 (week 5–6):** decommission the polling path in the customer
   app; reduce order-service read capacity accordingly.

**Backwards compatibility:** none required — this is a net-new endpoint.

**Rollback:** flip the customer-app feature flag back to off; clients
fall back to polling. The projections service can keep running (it's
read-only) or be scaled to zero. No data migration to undo.

**Data backfill:** before Phase 3, run a one-time full backfill: bump the
consumer group offset to earliest, let it grind through all retained
history (~15min at 24 partitions in parallel). Conditional-write CAS
makes this safe to repeat.

## 9. Observability & operations

Already detailed in §6 (Observability). Operational summary:

- **On-call rotation:** platform team's existing rotation.
- **Dashboards:** one Grafana dashboard `order-projections` showing lag
  per partition, DLQ rate by reason, apply duration histogram, read API
  RED, Spanner write conflicts.
- **Runbook items:**
  - Replay a single order: `ops/replay-order.sh <order_id>`.
  - Drain DLQ: `ops/replay-dlq.sh --reason <reason>`.
  - Consumer stuck on poison message: identify partition from lag
    metric, inspect offset, force-skip via offset reset (documented
    procedure, requires two-person approval).
  - Spanner regional failover: documented; tested quarterly.

## 10. Open questions

> Items marked **[BLOCKER]** are design-flipping; the answer determines
> whether the design as drafted is viable. Items marked **[IMPORTANT]**
> change implementation detail or operations. **[NORMAL]** is finer detail.

1. **[BLOCKER]** **Is this storage decision (Spanner vs DynamoDB) constrained
   by an explicit multi-cloud roadmap?** If yes → DynamoDB becomes viable
   despite cross-cloud latency. If no → Spanner. Currently designed as
   Spanner; would be a focused rewrite of §4i and §5 to swap. Recommendation:
   Spanner unless someone produces a real multi-cloud decision document.

2. **[BLOCKER]** **Does the producer emit a monotonic per-`order_id`
   `event_version`?** The entire dedup-and-reorder defense in §4b depends
   on it. If the order service doesn't currently emit it, adding it is an
   upstream change the order team has to commit to *before* this project
   starts. There is no good substitute (timestamps are not monotonic across
   clocks; offsets are per-partition, not per-order). Recommendation: hard
   prereq, escalate to order-service tech lead this week.

3. **[IMPORTANT]** **What is the order-service producer's idempotence
   config?** `enable.idempotence=true`, `acks=all`,
   `max.in.flight.requests.per.connection<=5`? If not, producer-side
   reordering is a real risk and we need to either fix it upstream or
   widen our pending-buffer window.

4. **[IMPORTANT]** **Are all `orders.events.v1` events truly partitioned by
   `order_id`?** The PRD says yes, but PM mentioned seeing cross-partition
   reorders during rebalances — that should be impossible for a
   single-key-partitioned topic, so something is off. Audit producer code
   in the order service and any other producer touching this topic.

5. **[IMPORTANT]** **Kafka retention on `orders.events.v1` — is it really
   7 days?** Our backfill / DR story depends on this. If it's 24h, we have
   to either lengthen retention or ship a separate event archive.

6. **[NORMAL]** **2-year growth target.** Assumed 5× (250K → 1.25M events/
   day). At that volume the design still holds; just confirming.

7. **[NORMAL]** **Right-to-be-forgotten / GDPR delete event:** does
   `customer_deleted` or `order_redacted` exist as an event type today?
   If not, the order service needs to emit it and we need to handle it
   (zero out PII fields in the projection, retain only the order_id and
   status).

8. **[NORMAL]** **Per-customer order list projection — when?** Out of scope
   for v1, but if the customer-app team needs it for Q4, design it now to
   share infra. Tentatively: separate Spanner table, separate consumer
   group on the same topic (or same consumer writing to both tables — TBD).

9. **[NORMAL]** **Auth header contract from the gateway.** The exact
   header name and signing scheme — copy from order service.

10. **[NORMAL]** **Cost approval for Spanner ($650/mo baseline).** Confirm
    with platform finance before commit.
