# TDD: Order Projections Service

**Author:** Tech Lead (drafted on behalf of the team)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-order-events-consumer.md`
  ("Real-time Order Status Projections", Marcus Webb, 2026-06-02)

---

## 1. Problem & context

The customer-facing app today polls the order service every 5 s for order
status, costing ~120 RPS of synthetic load against an OLTP system and
producing user-visible staleness of up to 5 s. We want a sub-second
"order status" feed without strangling the order service.

The order service already publishes domain events to Kafka
`orders.events.v1` (~14 event types, partitioned by `order_id`, 24
partitions). We will build a long-running consumer service that projects
those events into a denormalized per-order read model, plus a thin read
API behind the existing internal API gateway.

Consumers: the customer app (read API), and ops (replay tool). Source of
truth remains the order service; this is a derived read model.

Driver: customer-app team needs the feed live by **Sept 30 2026** for the
holiday-season ramp.

## 2. Scope

**In scope**
- Kafka consumer of `orders.events.v1` that maintains a per-order
  projection (status, line items, payment summary, fulfillment summary,
  refund summary).
- Storage of the projection in a low-latency store.
- `GET /orders/{order_id}/projection` behind the existing API gateway.
- Manual per-order replay tool for ops.
- Full back-fill from topic earliest (initial seed + recovery).
- DLQ for unprocessable events.

**Out of scope (explicit non-goals from PRD or deliberate cuts)**
- Replacing the order service as source of truth.
- Historical analytics (separate batch pipeline already exists).
- Per-customer order list / cross-order aggregations — these are a
  *second* projection on the same event stream and are explicitly
  deferred. The design leaves room for them (see section 11 — Open
  Questions and section 12 — Extensibility).
- Cross-region / multi-region serving. Single-region (the GKE region that
  hosts the order service).

## 3. High-level approach

A horizontally scaled Kafka consumer group on GKE consumes
`orders.events.v1`, applies each event through a deterministic state
machine, and writes the resulting projection to **DynamoDB Global Tables —
single region for v1**, using a conditional write keyed on the projection's
monotonic version. A small Spring Boot read service exposes
`GET /orders/{order_id}/projection` behind the existing API gateway,
reading the same DynamoDB table with eventually-consistent reads
(strongly-consistent reads are an opt-in per call). Unprocessable events
are routed to a DLQ topic with structured failure metadata.

Note: GCP-shop choosing DynamoDB is a deliberate deviation justified in
Section 5. If the org has a hard "no AWS" rule, the recommendation
collapses to Spanner; see Alternatives.

Diagram (prose):

```
[order service]
      |
      v
 [Kafka: orders.events.v1, 24 partitions, key=order_id]
      |
      v
 [order-projections consumer group on GKE, N pods]
      |   per-partition single-threaded apply
      |   conditional write by (order_id, expected_version)
      v
 [DynamoDB: order_projections]      [DLQ: orders.events.v1.dlq]
      ^
      |
 [order-projections read API on GKE]
      ^
      |
 [internal API gateway] <-- [customer app]
```

A separate `seed/replay` mode of the same binary, run as a GKE Job, drives
back-fills from a chosen offset for either the whole topic or a single
`order_id`.

## 4. Detailed design

### 4a. Broker & topic model

- **Broker:** Confluent Kafka on GCP (the PubSub-mirrored cluster). We
  consume Kafka, not the PubSub mirror, so we keep native Kafka
  semantics: per-partition ordering, consumer-group offsets, the
  Schema Registry. The mirror is a producer-side concern, not ours.
- **Topic:** `orders.events.v1`, 24 partitions, key = `order_id`.
  Retention: TBD with platform team — we need at least **7 days** of
  retention so a 1-week incident plus weekend triage can be recovered
  by replay; for initial seed we need topic compaction off and full
  history available (see Open Question 1).
- **Producer contract:** Avro on Confluent Schema Registry, BACKWARDS
  compatibility enforced (per PRD).
- **Consumer group:** `order-projections.v1`. One group for the
  read-model projection. Other future projections (e.g. per-customer
  list) get their own group on the same topic — that is the whole point
  of using a log.
- **Throughput envelope (provisional, for sizing only — see Open
  Question 2):** 50 K orders/day baseline; assume ~6 status transitions
  per order across the lifecycle = **~300 K events/day = ~3.5 events/s
  steady, ~50 events/s peak during flash sales**. Black Friday last
  year: 18 K orders/day → ~108 K events/day; the steady-state rate is
  well within a single consumer pod's headroom and the design is
  bottlenecked by burstiness during flash-sale traffic spikes and by
  full back-fill, not by steady-state.

### 4b. Delivery semantics & ordering — the part the team keeps getting bitten by

This is the core of the design. The PRD specifically calls out
out-of-order arrivals during broker rebalancing.

**Layer 1 — broker semantics (what we get):**
- At-least-once delivery. Duplicates are expected on consumer restart,
  rebalance, and producer retry.
- Per-partition ordering. Because the topic is keyed by `order_id`, all
  events for a single order land on a single partition and the broker
  *does* deliver them in producer-publish order on the happy path.

**Layer 2 — the failure cases that produced "out of order" complaints
from the team:**
1. **Rebalancing redelivery.** When a consumer leaves the group (deploy,
   crash, slow processing exceeding session timeout), the partition is
   reassigned and uncommitted records are re-delivered. From a naive
   consumer's perspective: "I just saw event 5, then event 3" — but
   that's not really out-of-order, it's redelivery of an older event.
   The fix is idempotency and an explicit version check, not "make
   Kafka ordered."
2. **Producer-side reordering on retry.** Without
   `max.in.flight.requests.per.connection=1` *and*
   `enable.idempotence=true` on the producer, a retry of event N can
   land after event N+1 in the same partition. This is on the order
   service, not on us — but we have to assume it can happen and we
   must verify (see Open Question 3).
3. **Cross-partition reordering** is a non-issue for a *single order*'s
   state because `order_id` keys the topic, but it *will* break any
   future projection that crosses orders (e.g. per-customer list).
   Flag for the second projection, not this one.

**Layer 3 — what this service guarantees (the "effectively-once"
construction):**
- We treat the stream as at-least-once.
- Every event in the Avro schema carries (or will carry — see Open
  Question 4) an `event_id` (UUID), `order_id`, an `occurred_at`
  timestamp, and a monotonically increasing per-order `sequence_no`
  produced by the order service. **If `sequence_no` is not in the
  schema today, getting it added is on the critical path and blocks
  this design**. Without it, we cannot detect out-of-order in a way
  that doesn't fall back on timestamps (which are not reliable across
  producer clocks). This is the most important upstream ask in the
  whole TDD.
- The projection row carries the `last_applied_sequence_no`.
- Apply rule:
    - If `event.sequence_no == projection.last_applied_sequence_no + 1` → apply.
    - If `event.sequence_no <= projection.last_applied_sequence_no` →
      duplicate or replay; **drop silently, increment a metric, log at
      debug**. This is the at-least-once dedup path. No DLQ.
    - If `event.sequence_no > projection.last_applied_sequence_no + 1` →
      we missed events. Options below.
- **The "gap" case** (we got event 7 but expected 5): two strategies,
  pick one (see Open Question 5 for the team to ratify — recommend B):
    - **A. Buffer-and-wait.** Hold event 7 in memory, do not commit the
      offset, wait for events 5 and 6. Risk: unbounded buffer if 5
      and 6 are genuinely lost; consumer lag grows; partition stalls.
    - **B. Apply-and-reconcile (recommended).** Apply event 7 (state
      machine accepts forward jumps for forward-only transitions like
      `cancelled`), record the gap in a `projection_gaps` table, and
      let the gap close itself when 5 and 6 arrive (their `sequence_no`
      check will pass because we track *seen* sequence numbers as a
      bitset / range list, not just the max). If they do not arrive
      within a configurable window (default 5 min), page on-call and
      fire a replay for that `order_id` from topic-earliest.
- **Conditional write to DynamoDB:** the apply is a `UpdateItem` with
  `ConditionExpression: last_applied_sequence_no = :expected`. If two
  consumer instances race (rebalance redelivery), one wins, the other
  gets `ConditionalCheckFailedException` → re-read, re-apply (or drop
  if the event is now stale). This is the *real* mechanism that makes
  the projection idempotent. The version check is in the database, not
  in the consumer's memory.

**Net guarantee:** at-least-once delivery + per-order sequence-checked
idempotent application + conditional write = **effectively-once
projection updates with bounded out-of-order tolerance**. We do not
claim Kafka exactly-once. We claim "the projection converges
deterministically and is safe to replay."

### 4c. Offset / checkpoint management

- **Commit timing:** after DynamoDB write succeeds AND state-machine
  apply returns. Never before. Commit-before-process is the
  at-most-once footgun named in the category reference.
- **Commit cadence:** **manual commit per batch of 100 records or 1 s,
  whichever comes first**. Per-message commit is too expensive at peak;
  per-batch is fine because the conditional write makes replays safe.
- **Session / rebalance config:**
    - `max.poll.interval.ms = 60000` — generous because peak per-event
      work is dominated by a single DynamoDB write (~10 ms p99) plus
      occasional state-machine validation.
    - `session.timeout.ms = 30000` with `heartbeat.interval.ms = 10000`.
    - `enable.auto.commit = false`. Commits are explicit.
    - Cooperative-sticky partition assignor — minimizes the
      stop-the-world rebalance pause that triggers the redelivery the
      team keeps seeing.
- **Replay:**
    - Per-order replay (ops tool): scans the topic from earliest for
      events matching `order_id`, replays through the same state
      machine into a *shadow* projection key, then atomically renames
      / overwrites. Acceptable because per-order replay is rare and
      bounded; reading the whole topic for one order is O(retention),
      which is fine at our volume.
    - Full back-fill (initial seed and disaster recovery): a separate
      `seed` consumer group on the same topic from `earliest`, run as
      a GKE Job with higher parallelism (24 pods, 1 per partition).
      Writes go into the *same* DynamoDB table; conditional writes
      make this safe to run alongside the live consumer because
      `last_applied_sequence_no` blocks regression.
- **Poisoned commit recovery:** because we commit after write, a
  poisoned-commit scenario (offset advanced past an unprocessed
  message) can only happen on a DLQ path; see 4d. To recover, ops can
  reset the consumer group offsets via the Kafka CLI to the
  earliest-uncommitted point; conditional writes prevent
  double-application.

### 4d. Poison-message / DLQ handling

- **Trigger:**
    - 3 retries with exponential backoff (100 ms, 500 ms, 2 s) on
      transient errors (DynamoDB throttling, Schema Registry 5xx,
      transient network).
    - Immediate DLQ on deterministic errors: schema deserialization
      failure, unknown event type, state-machine rejection
      ("`fulfilled` on never-created order" — PRD calls this out as
      explicitly DLQ-eligible).
- **DLQ destination:** Kafka topic `orders.events.v1.dlq`, same
  partition count, infinite retention (or 90 days — see Open
  Question 6). Payload wrapper:
  ```
  {
    original_topic, original_partition, original_offset,
    original_key, original_value_bytes (base64),
    failure_reason (enum), failure_message (string, no PII),
    consumer_attempt_count, first_failed_at, last_failed_at,
    schema_id
  }
  ```
- **Replay path:** ops dashboard (later phase) or a CLI that reads from
  the DLQ, lets a human edit the event (or just decide to skip), and
  re-injects to the main topic with the same partition key. Until that
  exists, the manual procedure is documented in the runbook.
- **Alarm thresholds:**
    - DLQ rate > 5 events / 5 min → page (probably a schema or
      producer bug, blast radius is wide).
    - DLQ count > 50 in 1 h → page.
    - Any DLQ for `event_type = order_created` → page immediately
      (these *must* succeed; if they don't, the entire order is
      invisible to the customer).

### 4e. Concurrency & parallelism

- **Topology:** N consumer pods × 1 consumer thread each. We keep one
  thread per pod and let Kafka's partition assignment do the
  parallelism — this trivially preserves per-partition ordering.
- **Pod count:** start with **6 pods**, giving 4 partitions per pod.
  Headroom: we can scale to 24 (one per partition) before adding
  partitions becomes necessary. **Never run more pods than partitions
  in steady state** (idle consumers, no throughput gain, and they slow
  rebalances).
- **Per-partition serialization:** strictly one event at a time per
  partition. No internal worker pool inside a partition — the moment
  we introduce one we lose per-partition ordering and the whole
  effectively-once story collapses.
- **In-flight downstream calls:** the only blocking call per event is
  DynamoDB `UpdateItem`. We use the async DynamoDB SDK with a per-pod
  semaphore of 32 in-flight requests (matches AWS SDK defaults; well
  below DynamoDB on-demand burst headroom).

### 4f. Back-pressure

- **Bottleneck order (slowest to fastest):**
    1. DynamoDB writes (p99 ~10 ms, throttle if > provisioned WCUs;
       on-demand handles burstiness up to 2× of previous peak).
    2. State-machine apply (in-process, sub-ms).
    3. Avro deserialize via Schema Registry (cached, sub-ms after warmup).
    4. Kafka poll (cheap).
- **What "can't keep up" looks like:** consumer lag grows. Lag is the
  primary health signal (see 9).
- **Hard ceiling:** Kafka retention. At 7 days retention, a 7-day
  outage = data loss. State this explicitly. We will request 14 days
  retention as a safety margin (see Open Question 1).
- **DynamoDB throttling response:** AWS SDK retries with exponential
  backoff; the consumer treats it as a transient error (see 4d) and
  goes through the 3-retry path. If a consumer is sustained-throttled,
  lag alerts fire and we switch the table to provisioned-with-autoscale
  or raise on-demand limits.

### 4g. Stateful processing

The consumer itself is **stateless** between events — all state lives
in DynamoDB. There is no in-memory window, no RocksDB, no local cache
that survives restart. This is a deliberate choice: it makes the
"single-pod failure" story trivial (Kafka reassigns the partition, the
new owner reads `last_applied_sequence_no` from DynamoDB on the first
event it processes, and continues).

In-memory caching of recent projections is allowed only as a hot-path
optimization for the *read* API, with a TTL no greater than 1 s; see
section 4i.

### 4h. Schema evolution

- **Compatibility mode:** BACKWARDS (per PRD). New consumer can read
  old events. We can add optional fields freely; we cannot remove or
  rename fields without a topic version bump.
- **Adding a new event type:** consumer enumerates known event types.
  An unknown type is **not** a poison message; it is logged at WARN,
  metric incremented, **skipped** (not DLQ'd). This is because new
  event types are expected during the period between order-service
  deploy and projections-service deploy; DLQ'ing them would create
  noise that masks real failures.
- **Breaking change rollout:** new topic `orders.events.v2`. Run both
  consumers in parallel. New projection key namespace. Cut read API
  over via flag. This is documented as the supported migration path,
  but no v2 is planned today.

### 4i. Storage choice — the principal-level take

The team is leaning DynamoDB. The user asked for an honest take. Here it
is. Each option below is evaluated on the four things that actually
matter for this workload: (1) read latency at p99, (2) cost of the
conditional-write idempotency story, (3) operational surface for a GCP
team, (4) cost.

**Recommendation: DynamoDB on-demand, single region (us-east-1 or the
region closest to the GCP region), single-table with PK = `order_id`.**
With a caveat — see "the GCP elephant" below.

**Schema:**
- Table: `order_projections`
- PK (HASH): `order_id` (string, UUID or ULID — confirm with order service).
- No SK. Single item per order — that's the projection.
- Attributes:
    - `last_applied_sequence_no` (number) — drives conditional write.
    - `status` (string) — denormalized for read.
    - `line_items` (list of maps).
    - `payment_summary` (map).
    - `fulfillment_summary` (map).
    - `refund_summary` (map).
    - `updated_at` (string, ISO8601).
    - `applied_sequence_ranges` (list of [from, to] number pairs) —
      tracks which sequence numbers we've actually applied so we can
      tolerate gaps (see 4b strategy B). For most orders this list is
      a single contiguous range `[[1, last]]`; only orders that hit
      the gap path grow it.
- **Item size:** estimated p99 ~8 KB (line items dominate). Well under
  the 400 KB DynamoDB hard limit. The largest orders we've seen are
  ~30 line items; even at 1 KB per item we're at 30 KB.
- **Capacity mode:** on-demand for v1. Reasoning: traffic is bursty
  (flash sales 3-4×), we have no production baseline, on-demand
  removes the most likely "we forgot to scale" incident. Migrate to
  provisioned-with-autoscale once we have 60 days of steady-state data
  and a cost model.
- **Indexes:** none for the consumer. The read API queries by
  `order_id` only. If the second projection (per-customer order list)
  needs `customer_id → [order_id]`, that is *not* an index on this
  table — that is a separate projection on the same event stream
  written to a separate table. Do not be tempted to put a GSI on
  `customer_id` here; it bakes the "many orders per customer" access
  pattern into the wrong projection.
- **TTL:** none. Projections are kept for the lifetime of the order
  service's own retention policy. If product policy is "delete
  projection 90 days after `cancelled`/`fulfilled`", that's a
  separate cleanup job (see Open Question 7).
- **Encryption:** SSE with AWS-owned KMS key for v1 (free, sufficient
  for non-PCI data; line items + customer IDs + addresses are PII so
  we may want a customer-managed key — see Open Question 8 and
  section 6 Security).

**The GCP elephant.** The platform is GCP. Putting DynamoDB in front of
a GCP service is a 4-step principal-level question:

1. **Cross-cloud latency.** GCP region (e.g. `us-central1`) ↔ AWS region
   (e.g. `us-east-4` / `us-east1`): ~10-15 ms p50 over public internet,
   ~3-8 ms p50 over Cloud Interconnect / Direct Connect. A 50 ms p99
   read budget tolerates one ~15 ms hop. We have to verify that the
   network path exists — running DynamoDB from GCP across the public
   internet is a non-starter for both cost (egress) and latency.
2. **Egress cost.** GCP → AWS egress is expensive (~$0.08-$0.12/GB
   depending on region). At our volume (50 K orders/day × 8 KB write
   × maybe 30 reads/order/day = ~13 GB/day), this is **~$30-50/month**
   for the write side and **~$30-50/month** for read. Fine. Not fine
   if reads spike 100× on a marketing campaign.
3. **Org capability.** Does the org actually have an AWS account, IAM
   roles to GCP service accounts (via Workload Identity Federation),
   on-call runbooks for AWS outages? If "no" to any of those, the
   recommendation must be Spanner.
4. **Data residency / compliance.** If there is a "no customer data
   leaves GCP" rule for any reason (contracts, certifications), this
   collapses to Spanner immediately.

**If GCP-only is required**, the answer is **Spanner** with the same
schema (single-row per `order_id`, primary key `order_id`,
`last_applied_sequence_no` as the conditional column, `UPDATE ... WHERE
last_applied_sequence_no = :expected` for idempotency). Spanner gives
us strong consistency by default, scales horizontally, costs more
than DynamoDB at low volume but is *cheaper than DynamoDB once we
account for cross-cloud egress and operational overhead*. At our
volume, Spanner is comfortable on a 1-node regional instance.

**My principal-level take:** if "GCP-only" is even slightly the org
norm, choose Spanner and stop. The "DynamoDB is cheaper" argument
ignores the cross-cloud surface area. The "DynamoDB is faster" argument
is wrong at any scale once you add a cross-cloud hop. The team should
only choose DynamoDB if (a) the org already has AWS in production for
other workloads, (b) there is an existing Interconnect, and (c) the
team has on-call coverage for AWS. Otherwise: **Spanner**.

I've left the recommendation as "DynamoDB" only because the PRD says
the team is leaning that way. I would push back in review.

### 4j. Read API

A thin Spring Boot (Kotlin) service exposes
`GET /orders/{order_id}/projection` behind the existing internal API
gateway. Single endpoint. Eventually-consistent DynamoDB read by
default; clients can request strongly-consistent via a header
(`X-Consistent-Read: true`) — costs 2× RCUs, used by ops tools only.

- **Caching:** in-process Caffeine cache, max 10 K entries, TTL 1 s.
  Bounded by the p95<2s end-to-end target — 1 s cache is fine because
  the contract is "sub-second after the cache window."
- **Response:**
    - 200 — projection JSON, with `last_applied_sequence_no` and
      `updated_at` echoed so clients can detect staleness.
    - 404 — order_id not seen yet. Important: 404 from this service
      does NOT mean "no such order" — it means "no projection yet".
      Document this in the API spec so the customer app doesn't show
      "Order not found" to a user whose order was just placed 200 ms
      ago.
    - 503 — DynamoDB unavailable; clients should retry.
- **Auth:** delegated to the existing internal API gateway (per PRD).
  The service trusts a `X-User-Id` header injected by the gateway and
  enforces "user can only read their own orders" via a
  `customer_id == X-User-Id` check on the projection. (Open Question 9:
  is `customer_id` on the projection from day 1? It should be.)

## 5. Alternatives considered

### Storage

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **DynamoDB on-demand** (recommended in PRD) | Cheap at low volume, well-known idempotency idioms, on-demand handles bursts. | Cross-cloud from GCP: latency hop, egress cost, AWS surface area in a GCP org. | Recommended **only if** org already on AWS. |
| **Spanner** | GCP-native, strong consistency by default, horizontal scale, single-region or multi-region with the same code. | Higher floor cost (~$650/mo per node); team needs to learn Spanner idioms. | Recommended for GCP-only orgs. **My pick on tradeoffs.** |
| **Postgres (CloudSQL)** | Team knows it; transactions are trivial; `UPDATE ... WHERE sequence_no = :expected` is one line. | Vertical scaling ceiling; 50 K orders/day is fine but the "per-customer projection" follow-up changes the picture; MVCC bloat under heavy update load is real (this workload is update-dominated). | Viable for v1, but the second projection forces a re-platform. Pass. |
| **Redis (Memorystore)** | Lowest read latency. | Not durable enough as source-of-truth for a read model that has to survive a restart with correct state. Redis-with-AOF is an option but the operational story is worse than DynamoDB or Spanner. | Use as a *cache* in front of the durable store, not as the store. Not in v1. |

### Consumer pattern

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Kafka consumer group, per-partition single-thread, conditional write** (recommended) | Preserves ordering, idempotent at the database, stateless consumer. | We have to get a `sequence_no` into the event schema. | Recommended. |
| **Kafka Streams** | Built-in state stores, per-partition processing, exactly-once available. | Adds RocksDB + state-store recovery; bigger ops surface; harder to back-fill into an external DynamoDB/Spanner table. | Pass; the projection lives in an external store, not in Streams state. |
| **Flink** | Best-in-class for exactly-once + windowed processing. | Massive ops surface for one stateless projection. Wrong tool. | Pass. |
| **Read-side projection (project on read)** | Zero new infrastructure. | Hammers the order service on every read — exactly what we're moving away from. | Pass. |

### Do nothing
Keep polling. Rejected: the polling load is already a known cost and
the PRD's sub-second target cannot be met by 5 s polling. The cost of
*not* doing this rises linearly with traffic.

## 6. Non-functional requirements

Walking `references/nfrs-checklist.md` in order.

### Scalability
- **Capacity target.** Steady-state ~50 K orders/day → ~300 K events/day;
  peak ~5× steady (250 K orders/day during flash sales,
  ~1.5 M events/day, ~17 events/s). Black Friday 2026 we should plan
  for 2× last year = ~36 K orders/day → comfortable.
- **Scaling axis.** Horizontal on consumer pods up to 24 (one per
  partition). To go further, we add partitions (offline operation; needs
  producer cooperation). DynamoDB / Spanner scale independently.
- **Stateful bottleneck.** None in the consumer (stateless). DynamoDB
  is bounded by per-partition-key write throughput (1 KB/s for a single
  PK in on-demand) — since our PK is `order_id`, hot-partition risk is
  per-order, and a single order does not exceed 50 events in its
  lifetime. Spanner has no per-row throughput cap relevant here.
- **Back-pressure.** Lag grows; alerts fire (see 9). Last-resort
  back-pressure: pause consumer, let the topic act as the buffer, until
  retention runs out.
- **Verification.** Load test at 5× peak (~100 events/s sustained for
  30 min) before launch; back-fill drill at 10× sustained.

### Reliability & availability
- **Availability SLO.** Read API: 99.9% monthly. Consumer: "no message
  loss" is the real SLO; we tolerate up to 5 min of staleness during
  recovery (one rebalance cycle). DLQ rate budget: < 0.1% of events,
  rolling 24 h.
- **SPoFs.** GKE cluster (mitigated by multi-zone node pools); Kafka
  cluster (Confluent's problem); DynamoDB/Spanner (managed, multi-zone).
  Schema Registry: single point — cache schemas locally in the consumer
  to survive a 1 h Registry outage.
- **Dependency failure handling.** DynamoDB: SDK retries with backoff,
  3-attempt cap, then DLQ-as-transient (re-attempted on next poll, not
  permanently DLQ'd). Schema Registry: local cache; on miss, retry with
  backoff; if a *new* schema can't be fetched, treat as transient.
  Kafka: consumer client handles reconnection; we never block on the
  broker for application logic.
- **Idempotency.** Per-order `sequence_no` + DynamoDB conditional write
  (or Spanner `UPDATE WHERE ... = :expected`). This is the load-bearing
  reliability mechanism in the design — section 4b.
- **Verification.** Chaos test: kill a consumer pod mid-batch and
  verify no `last_applied_sequence_no` regression. Replay test: full
  back-fill into a live table should produce identical state.

### Resilience
- **Graceful degradation.** Read API: if DynamoDB is slow, the Caffeine
  cache serves slightly-stale reads (up to 1 s). If DynamoDB is fully
  down, 503 with `Retry-After: 5`. The customer app should fall back to
  the old polling path (which still exists in v1 — see rollout).
- **Bulkheads.** Read API and consumer are separate deployments; a
  consumer hot loop cannot starve the read API.
- **Timeouts.** DynamoDB SDK: 1 s connect, 3 s call. Schema Registry: 2 s.
  Kafka poll: 500 ms. No unbounded waits.
- **Backoff & jitter.** Exponential backoff with full jitter
  (100 ms / 500 ms / 2 s base) on all retries.

### Performance
- **Latency targets.**
    - Read API: **p99 < 50 ms** (per PRD).
    - End-to-end producer→projection: **p95 < 2 s** (per PRD).
- **Resource budget.** Consumer pod: 1 vCPU, 1 GB RAM. Read API pod:
  0.5 vCPU, 512 MB RAM.
- **N+1.** None. One DynamoDB read per API call; one DynamoDB write
  per event.
- **Verification.** Load test (see Scalability). Continuous SLO
  monitoring post-launch.

### Observability
- **Metrics.**
    - `kafka_consumer_lag_records{partition}` — gauge, alert at lag >
      5 K records for 5 min OR lag > 30 min of wall time.
    - `events_processed_total{event_type, outcome}` — counter, outcomes
      `applied | duplicate | gap_applied | dlq | retried`.
    - `dynamodb_write_duration_seconds{outcome}` — histogram.
    - `dynamodb_conditional_check_failed_total` — counter, should be
      near-zero except during rebalances; alert on sustained > 1/s.
    - `schema_deserialize_failures_total{schema_id}` — counter.
    - `gap_orders_open_total` — gauge of orders currently in the "gap
      seen, missing events not yet arrived" state.
    - Read API: standard RED — `http_requests_total{route, code}`,
      `http_request_duration_seconds{route}`.
- **Logs.** Structured JSON. Include `order_id`, `event_id`,
  `sequence_no`, `partition`, `offset`, `trace_id`. Never log full
  line items (PII).
- **Traces.** OpenTelemetry. Sample 1% of consumer events; 100% of
  read API requests with `X-Debug: true` header.
- **Alerts.**
    - Page: consumer lag > 30 min wall-time, DLQ rate > 5/5 min,
      `order_created` event in DLQ, read API p99 > 100 ms for 5 min,
      DynamoDB sustained throttling.
    - Ticket: gap_orders_open_total > 100, schema deserialize failures
      > 10/h on a single schema.
- **Runbook.** `<link to runbook — to be added>`. Covers: lag
  spike investigation, DLQ replay procedure, per-order replay,
  back-fill execution.

### Security
- **AuthN.** Read API: trusts gateway-injected `X-User-Id` header.
  Consumer: SASL/SCRAM to Kafka, IAM role (or Workload Identity
  Federation if DynamoDB on AWS) to the store.
- **AuthZ.** Read API enforces `projection.customer_id == X-User-Id`.
  No admin endpoints.
- **Secrets.** Kafka SASL credentials, AWS credentials (if DynamoDB)
  in GCP Secret Manager, rotated quarterly, mounted via CSI driver.
- **Data classification.** Projection contains PII (line items can
  include shipping address fragments, customer references). Treat as
  PII:
    - Encryption at rest: KMS-managed (customer-managed key —
      Open Question 8 for which).
    - Encryption in transit: TLS to store, TLS to Kafka.
    - Audit logging: store-level audit logs enabled (DynamoDB
      Streams + CloudTrail or Spanner audit logs).
    - Retention / deletion: aligned to order-service policy; not
      independent.
- **Input validation.** Avro deserialization is the validation
  boundary for events. HTTP request validation on the read API
  (path param format).
- **Threat model.** Internal-only. Behind the API gateway. Topic is
  internal Kafka. Cross-cloud surface area is the new threat
  (DynamoDB option only).
- **Supply chain.** Container images from internal registry, signed,
  pinned by digest. Gradle dependency lockfile.

### Cost
- **Estimated unit cost.**
    - Consumer pods (6 × 1 vCPU / 1 GB): ~$80/month on GKE.
    - Read API pods (3 × 0.5 vCPU / 512 MB): ~$25/month.
    - DynamoDB on-demand: write ~$0.0013/1K writes × 300K/day = ~$12/mo;
      read ~$0.00025/1K reads × 1.5M/day (assume 30 reads/order) =
      ~$11/mo. Cross-cloud egress: ~$60/mo. **DynamoDB total ~$85/mo.**
    - Spanner (1 regional node): ~$650/mo (+ storage ~$10/mo). **Spanner
      total ~$660/mo.**
- **Cost drivers.** Spanner: the node count. DynamoDB: egress to GCP,
  and write throughput at flash-sale peaks.
- **Cost ceiling.** Page if monthly bill exceeds 2× the estimate.

### Compliance & privacy
- **Data residency.** Project assumption: customers are not subject to
  EU residency rules (Open Question 10). If they are, DynamoDB option
  is OUT and Spanner config must specify region.
- **Retention.** Aligned to order-service retention. No independent
  policy on the projection.
- **Regulatory.** GDPR right-to-erasure: when the order service hard-
  deletes an order, we must propagate. Mechanism: order service emits
  a `order_purged` event; consumer hard-deletes the projection row.
  No PCI exposure (payment summary, not card data).
- **Audit log.** Store-level audit logs as above.

### Deployment & operations
- **Deploy strategy.** Rolling deploy on GKE, 1 pod at a time, with a
  60 s readiness gate after Kafka group join. **No canary on the
  consumer**: canarying a partitioned consumer is meaningless because
  partition ownership is a group decision, not a pod decision.
- **Rollback.** Roll the deployment back; consumer offsets are
  unchanged so we pick up where we left off. Projection rows are
  forward-compatible by construction (additive fields only — schema
  rules above).
- **Feature flags.** `projections.read.enabled` on the customer app
  side gates whether reads hit this service or the old polling path.
  Default off at launch; ramp by percentage.
- **Config.** Helm values + ConfigMap; secrets in Secret Manager.
  Change requires PR + deploy; no live config edits.
- **Migrations.** Schema migrations on DynamoDB are runtime additions
  of attributes — zero downtime. Spanner: any DDL is online, zero
  downtime within reason.

### Disaster recovery
- **RTO / RPO.**
    - **RPO: 0** for the projection (we can always recompute from the
      Kafka topic, assuming retention covers the window).
    - **RTO: < 4 h** for full back-fill of all orders from topic
      earliest, dominated by Kafka throughput and DynamoDB/Spanner
      write rate. Verified by drill.
- **Backups.**
    - DynamoDB: PITR enabled (35 days), continuous backup.
    - Spanner: managed backups, daily, 30 days retention.
    - The *real* backup is the Kafka topic. Verify Kafka backup /
      retention with platform team.
- **Multi-region story.** None for v1. Single region. If the region
  goes down, customer app falls back to the (still existing) polling
  path. Multi-region is a v2 conversation tied to the order service's
  own DR story, not this projection.

### Maintainability
- **Test strategy.**
    - Unit: state-machine transitions (every event type × every state
      → expected outcome), sequence-number dedup logic, gap handling.
    - Integration: testcontainers Kafka + DynamoDB Local (or Spanner
      emulator); end-to-end produce → consume → query.
    - Contract: Avro schema compatibility check in CI against the
      Schema Registry.
    - Load: see Performance.
    - Chaos: pod-kill in staging; broker rebalance forced; DynamoDB
      throttling injected.
- **Documentation.** TDD (this doc), API spec (OpenAPI), runbook.
- **Ownership.** Owning team: customer-platform (TBD — Open Question 11).

### Extensibility
- **Variation points.** New event types: enumerated in state machine,
  unknown types are skipped not failed (see 4h). New projections on
  the same stream: new consumer group, new table. New read endpoints:
  add to read API; do not add GSIs to the projection table.
- **Versioning.** Avro schemas: BACKWARDS-compatible. Read API:
  URL-versioned (`/v1/orders/...`). Projection schema: forward-only,
  additive.

## 7. Risks & failure modes

| Risk | Likelihood | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Producer emits events out of order without `sequence_no` | Medium (depends on producer config we don't yet know) | All orders affected by reordered window | Confirm producer config; require `sequence_no` in schema; gap-tolerant apply rule. | Replay affected order from topic-earliest. |
| Kafka outage > retention window | Low | Up to N days of orders unreadable from projection | 14-day retention ask; multi-AZ Confluent cluster. | Back-fill from order service (out-of-band script). |
| DynamoDB throttling during flash sale | Medium-low (on-demand handles it) | Lag spike, eventual catch-up | On-demand mode; alert on throttle metric. | Scale DynamoDB; consumer catches up. |
| Schema Registry outage | Medium | New event types fail to deserialize | Local schema cache. | Fall back to cached schemas; alert. |
| Poison message blocks partition | Low (we DLQ on deterministic errors) | 1 partition (1/24 of orders) | DLQ on deterministic errors after 3 retries on transient. | DLQ replay tool. |
| Cross-cloud egress cost spike (DynamoDB option only) | Medium | Surprise bill | Cost alerting. | Move reads behind cache; consider Spanner migration. |
| Gap that never closes (lost events) | Low | One order is stuck mid-state | Gap timeout (5 min) + page. | Per-order replay from topic-earliest. |
| Rebalance storm under deploy | Medium | Lag spike during deploy window | Cooperative-sticky assignor; rolling deploy with readiness gate. | Wait it out; alert if > 5 min. |
| Customer app reads projection 100 ms after order placed → 404 | High (this WILL happen) | One customer sees "order not found" briefly | 404 means "not yet" not "never"; document; customer app must handle. | None needed; resolves in < 2 s p95. |

## 8. Rollout & migration

- **Phase 0 (week -2 to 0):** Schema change request to order service —
  add `sequence_no` if missing. **This is the long-pole dependency.**
- **Phase 1 (week 1-3):** Build consumer + read API behind a feature
  flag, default off. Deploy to staging, run synthetic load.
- **Phase 2 (week 4):** Production back-fill from topic earliest.
  Verify projection completeness against order service for a 1%
  sample.
- **Phase 3 (week 5-6):** Customer app integrates against the new read
  API, but still calls the polling path. Both paths return; customer
  app compares for divergence and logs. **Shadow mode.**
- **Phase 4 (week 7-8):** Flip flag to 1% → 10% → 50% → 100% of
  customer-app reads over a week. Keep the polling path warm for 30
  days.
- **Phase 5 (week 9):** Decommission the polling path.

**Rollback:** flip the flag back. Polling path serves traffic. No
data migration to undo because the projection store is a derived
read model.

## 9. Observability & operations

Covered in detail in NFRs/Observability above. The on-call runbook
needs to cover, at minimum:

1. Lag spike — which partition, which consumer pod, downstream
   throttling check, restart procedure.
2. DLQ replay — manual procedure until tooling exists.
3. Per-order replay — ops CLI.
4. Full back-fill — GKE Job manifest, expected duration, monitoring
   queries.
5. Schema Registry outage — fall-back behavior, manual schema cache
   refresh.
6. DynamoDB / Spanner outage — graceful degradation behavior, customer
   communication.

Owning team / on-call rotation: TBD (Open Question 11).

## 10. Multiple-workload note

This TDD covers an event-driven consumer **plus** a small read API.
The consumer is the dominant workload by complexity and risk; the read
API is a thin layer on top of the same data. Treating them as one TDD
is appropriate because they share the data store and lifecycle. If the
read API grows non-trivial logic (per-customer aggregation, ranking,
search), it becomes its own TDD.

## 11. Open questions

The eval substitution rule applies to these — the user is not
reachable here, so each item records a working assumption and the
information needed to ratify it.

1. **Kafka retention.** What is the current retention on
   `orders.events.v1`? Assumption: 7 days; design asks for 14.
   **Owner: platform team.**
2. **Event volume.** What is the actual events-per-order ratio
   today? Assumption: ~6. Drives consumer / DynamoDB capacity sizing.
   **Owner: order-service team.**
3. **Producer config.** Does the order service run with
   `enable.idempotence=true` and
   `max.in.flight.requests.per.connection=1`? If not, producer-side
   reordering is real and we have additional dedup work.
   **Owner: order-service team.**
4. **`sequence_no` in schema.** Does today's Avro schema have a
   monotonically increasing per-order `sequence_no`? Assumption: no,
   and adding it is on the critical path. **Owner: order-service
   team.** *This is the highest-risk dependency in the design.*
5. **Gap strategy.** Buffer-and-wait vs apply-and-reconcile. Design
   recommends apply-and-reconcile (4b). **Owner: this team to
   ratify; surface in design review.**
6. **DLQ retention.** Assumption: 90 days. **Owner: this team.**
7. **Projection deletion.** Do we have a "delete projection N days
   after terminal state" requirement? Assumption: no; aligned to
   order-service retention. **Owner: product.**
8. **KMS key.** Customer-managed key vs platform-managed?
   Assumption: customer-managed for PII alignment. **Owner: security.**
9. **`customer_id` on projection.** Required from day 1 for the
   read API auth check. Assumption: yes. Confirm.
   **Owner: order-service team.**
10. **Data residency.** Are we subject to EU-residency constraints?
    Assumption: no. If yes, DynamoDB option is out.
    **Owner: legal / compliance.**
11. **Ownership.** Which team owns this service in 12 months?
    Assumption: customer-platform team. **Owner: engineering
    leadership.**
12. **Storage final choice.** DynamoDB vs Spanner — see section 4i.
    Recommendation: Spanner unless the org is already on AWS in
    production. **Owner: this team + platform leadership; surface
    in design review.**

## 12. Future work — second projection (per-customer order list)

Explicitly out of scope but called out by the PRD. Notes for whoever
builds it next:

- Separate consumer group on the same topic.
- Separate table, keyed by `customer_id` with `order_id` as sort key
  (DynamoDB) or `(customer_id, order_id)` PK (Spanner).
- Cross-partition ordering is now a real problem: the per-customer
  view does *not* benefit from the per-`order_id` partition key. The
  consumer must accept that updates to two different orders for the
  same customer can arrive in either order; the view's contract has
  to allow that (each row carries its own version).
- Do not bolt a GSI onto the existing projection table to serve this.
  It's a different access pattern and a different consistency
  contract.
