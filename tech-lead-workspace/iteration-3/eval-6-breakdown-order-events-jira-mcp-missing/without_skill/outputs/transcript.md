# Order projections — ticket breakdown

Heads up before I dive in: **I don't have a Jira MCP / Jira API connection
wired up in this session**, so I can't actually create issues in your Jira
instance from here. What I've done instead is produce a ready-to-paste
ticket breakdown derived from the TDD. Once you give me Jira access (MCP
server, API token + project key, or a CSV import target), I can push these
in directly without re-deriving them.

Below: an epic + 12 stories, ordered for delivery, each with scope,
acceptance criteria, dependencies, and a rough size. I've kept them
outcome-oriented (no implementation code) and tied each back to the TDD
section it implements.

---

## Epic

### EPIC: Order projections service (canonical materialized view)

**Goal.** Replace per-team local order projections with a single canonical
projections service that consumes `orders.events.v2`, maintains a
Postgres-backed materialized view, and exposes a synchronous read API —
with effectively-once semantics, DLQ handling, and replay support.

**Source:** `evals/tdd-order-events-consumer.md`
**Success metrics (from TDD §6):**
- Sustained 200 ev/s, peak 1000 ev/s with p95 consumer lag < 30s
- Read API p99 < 100ms, 99.9% availability
- Zero per-team projection drift after cutover

---

## Stories (in delivery order)

### 1. Bootstrap `order-projections` service skeleton
**TDD ref:** §3, §4.1
**Scope.** New deployable service: repo, CI, base image, config loading,
health endpoints, structured logging, metrics scaffolding. No Kafka or DB
wiring yet.
**Acceptance criteria.**
- Service builds, deploys to staging, serves `/healthz` and `/readyz`.
- Structured logs include service name, version, env.
- Prometheus `/metrics` endpoint live (empty metrics OK).
- Config loaded from env vars; documented in repo README.
**Dependencies.** None.
**Size.** S.

### 2. Postgres schema for projection + idempotency log
**TDD ref:** §4.2
**Scope.** Create `order_projections` and `processed_events` tables with
the columns/PKs called out in TDD §4.2. Migrations checked in and run in
staging.
**Acceptance criteria.**
- Migrations apply cleanly forward and backward.
- `processed_events` PK is `(partition, offset)`; `order_projections` PK
  is `order_id`.
- Index plan documented; no production data touched yet.
**Dependencies.** #1.
**Size.** S.

### 3. Kafka consumer wiring with manual offset commit
**TDD ref:** §4.1
**Scope.** Connect to `orders.events.v2` with consumer group
`order-projections-v1`. Manual commit after successful transaction.
Auto-rebalance, mTLS to broker.
**Acceptance criteria.**
- Consumer joins group, claims partitions, logs assignment changes.
- mTLS handshake verified in staging.
- Offsets only commit after the downstream transaction commits (proven
  by integration test that crashes mid-handler and shows no offset
  advance).
**Dependencies.** #1.
**Size.** M.

### 4. Event schema + Avro deserialization with schema registry
**TDD ref:** §7 (schema evolution)
**Scope.** Register/consume Avro schemas for `order.{created,paid,
shipped,cancelled,refunded}`. CI guard for BACKWARD compatibility.
**Acceptance criteria.**
- Consumer rejects (sends to DLQ — pending #7) any event that fails
  schema validation.
- CI fails on a backward-incompatible schema change.
- Schemas documented in repo.
**Dependencies.** #3.
**Size.** M.

### 5. Idempotent projection apply (transactional)
**TDD ref:** §3, §4.2
**Scope.** For each event: in a single Postgres transaction, check
`processed_events` for `(partition, offset)`, apply state transition to
`order_projections`, insert idempotency record, then commit. Skip if
already applied.
**Acceptance criteria.**
- Replaying the same event twice produces exactly one apply (verified by
  test).
- Crash between transition apply and offset commit leaves no
  inconsistency on restart.
- Unit + integration tests for each event type's transition.
**Dependencies.** #2, #3, #4.
**Size.** L.

### 6. Out-of-order protection via `event_seq`
**TDD ref:** §4.3
**Scope.** Track `last_event_seq` per order; reject events with
`event_seq <= last_event_seq` as stale (no error, counted in a metric).
**Acceptance criteria.**
- Replayed historical event for an order in a newer state is silently
  dropped and counted in `events_consumed_total{result="stale"}`.
- Forward-progress events still apply.
- Test covers producer-replay scenario from TDD §4.3.
**Dependencies.** #5.
**Size.** S.

### 7. DLQ for poison events
**TDD ref:** §4.4
**Scope.** After 3 in-process retries, publish failed events to
`orders.events.v2.dlq` with failure metadata. Consumer continues past
poison events; partition does not block.
**Acceptance criteria.**
- Schema-invalid, undeserializable, and invalid-transition events end up
  on the DLQ topic with original key/value + failure reason header.
- Retry count and DLQ emissions visible in metrics.
- Integration test injects each failure class and asserts DLQ
  publication + consumer progress.
**Dependencies.** #5.
**Size.** M.

### 8. Read API: `GET /orders/{order_id}`
**TDD ref:** §4.5, §6 (latency)
**Scope.** Synchronous HTTP endpoint returning the current projection.
Internal-only auth using existing platform middleware. Connection pool
sized for target QPS.
**Acceptance criteria.**
- p99 < 100ms under target read load in a load test.
- 404 for unknown order_id; 200 with full projection payload otherwise.
- AuthN/AuthZ enforced per platform standard.
**Dependencies.** #5.
**Size.** M.

### 9. Observability: metrics, logs, dashboards, alerts
**TDD ref:** §9
**Scope.** Implement the metrics named in TDD §9; ship a Grafana
dashboard; configure alerts: lag > 30s/5min, DLQ rate > 0.1%/10min,
rebalance storms (>3/5min), read API error rate > 1%.
**Acceptance criteria.**
- All TDD §9 metrics emitted and labeled correctly.
- Dashboard merged and linked in runbook.
- Alerts firing in staging when conditions are synthesized.
**Dependencies.** #5, #7, #8.
**Size.** M.

### 10. Back-fill / replay tool
**TDD ref:** §8 (rollout), §7 (drift mitigation)
**Scope.** Operator tool that runs the consumer logic on a separate
throttled consumer group from a chosen offset/timestamp. Reuses
idempotency log so it is safe to run alongside the live consumer.
**Acceptance criteria.**
- Replay from earliest retention completes and converges projection
  state to live consumer's state.
- Write throttle is configurable; default protects Postgres per TDD §7.
- Documented runbook entry for "cold-start replay".
**Dependencies.** #5, #6.
**Size.** M.

### 11. Shadow-mode rollout + parity check
**TDD ref:** §8
**Scope.** Run the consumer in production with the read API disabled.
Build a parity job that samples orders and compares our projection
against each per-team local projection for one week.
**Acceptance criteria.**
- Read API gated behind `projection_api_enabled` feature flag (off in
  prod).
- Parity job runs daily, reports diffs in a dashboard.
- Sign-off criteria for cutover documented (e.g. <0.01% diffs for 7
  consecutive days).
**Dependencies.** #5, #6, #7, #9, #10.
**Size.** M.

### 12. Cutover + per-team opt-in
**TDD ref:** §8
**Scope.** Enable `projection_api_enabled` per consuming team; each team
migrates reads off their local projection. Keep local projections live
for 30 days post-cutover for rollback.
**Acceptance criteria.**
- Each consumer team's migration tracked as a sub-task with sign-off.
- Rollback procedure verified in a game day.
- 30-day deprecation timer started; tracked to deletion.
**Dependencies.** #11.
**Size.** M (coordination-heavy).

---

## Open questions to resolve before sprinting (TDD §10)

These aren't tickets yet — they gate scope on #2 and a future story:

- **OQ1: v2 → v3 schema replay strategy.** Need orders-team sign-off on
  translator-job approach. Blocks any future breaking-schema work; does
  not block #1–#12.
- **OQ2: customer-level read SLO.** If sub-100ms customer lookups are in
  scope, add a story for a `customer_id` secondary index + a
  `GET /customers/{customer_id}/orders` endpoint. Affects #2 schema and
  adds one story after #8.

I'd suggest pinning both to the epic and resolving OQ2 before sprint
planning so #2's schema doesn't get rev'd mid-build.

---

## To actually get these into Jira

Pick whichever is easiest on your side and I'll run with it:

1. **Stand up a Jira MCP server** (e.g. the Atlassian MCP) and re-invoke
   me — I'll create the epic + 12 stories with the structure above.
2. **Give me a Jira API token + base URL + project key** and I can call
   the REST API directly.
3. **CSV import.** Say the word and I'll emit a Jira-CSV-import-ready
   file in this same shape (Epic Link column wired up), which you can
   import via Jira's CSV importer in one go.

Let me know which path you want.
