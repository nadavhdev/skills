# TDD: Referral Code API

**Author:** Tech Lead (on behalf of Growth Eng)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-referral-api.md` (Sarah Chen, 2026-06-04)
**Workload category:** API serving (synchronous REST). Secondary surface: an
internal credit-issuance side-effect that is invoked from the existing
activation pipeline — designed here but explicitly not its own service.

---

## 1. Problem & context

The consumer mobile app (~2.1M MAU, +8% MoM) has no referral program. Growth
wants every existing user to be able to share a 6-char code; new users who
install via the code (deep link or manual entry) get attributed to the
referrer, and at first paid-subscription start both sides receive a $5
billing credit. Marketing needs reporting (`code → install → activation`)
for the August 25 back-to-school campaign. Mobile needs the API contract by
**July 1**.

Consumers of this system:

- Mobile app (iOS / Android), authenticated user sessions — reads/writes
  their own code, redeems a code at signup.
- Activation pipeline (existing internal service that flips a user to
  "first paid subscription started") — emits a signal we react to.
- Stripe + the existing internal credit ledger in Postgres — we *write*
  $5 credits into the ledger; we do not directly call Stripe for credit
  issuance (the ledger does, as today).
- Looker / Snowflake — read replica or CDC of our tables; owned by data
  team.

Why now: hard marketing deadline tied to back-to-school. Backend contract
freeze July 1, code freeze approximately August 11 to allow store review.

## 2. Scope

**In scope**

- New service module `referrals` (FastAPI, Python — matches team stack)
  exposing the user-facing referral endpoints.
- Postgres schema for codes, redemptions, and idempotent credit-issuance
  receipts.
- Deep-link resolution endpoint used by the install-attribution flow.
- A subscription to the existing `subscription.activated` domain event
  (whatever the current activation pipeline emits — see Open Q3) that
  performs credit issuance into the existing credit ledger.
- Fraud guardrails sufficient for launch (per-IP / per-device caps,
  velocity rules, manual-review queue). Not a full anti-fraud system.
- Observability, runbook, rollout/rollback plan.

**Out of scope**

- Multi-tier referrals.
- B2B / enterprise referrals.
- Building a new attribution SDK — we use whatever the mobile SDK already
  gives us for install attribution (AppsFlyer / Branch / Firebase — see
  Open Q1).
- Looker/Snowflake dashboarding — data team owns it from a read replica.
- Code-expiry mechanics (PRD says "currently assuming no").

## 3. High-level approach

A small FastAPI service owns three first-party concerns:

1. **Code lifecycle** — generate, fetch, validate. One persistent
   `referral_code` per user, 6 chars from a 32-char alphabet (Crockford
   base32 minus `0/O/I/1/L/U`).
2. **Redemption attribution** — when a new user signs up (mobile calls
   `POST /v1/referrals/redemptions` with their fresh user id and a code, or
   the deep-link resolver fires the same call server-side), we write a
   `pending` redemption row binding `referee_user_id → referrer_user_id`.
   At most one redemption per referee, enforced by a unique constraint.
3. **Credit issuance on activation** — we consume the existing
   `subscription.activated` event, look up the pending redemption for
   that user, mark it `activated`, and call the existing credit-ledger
   write API twice (once for referrer, once for referee), each with a
   deterministic idempotency key.

Diagram (prose):

```
[Mobile app] ──HTTPS──▶ [API Gateway / ALB] ──▶ [referrals-api (FastAPI)]
                                                  │
                                                  ├──▶ [Postgres: referrals schema]
                                                  │
                                                  └─◀── [subscription.activated event]
                                                          (from existing activation service,
                                                           consumed via SQS or the existing
                                                           internal event bus — see Open Q3)
                                                  │
                                                  ▼
                                          [credit-ledger internal API]
                                                  │
                                                  ▼
                                              [Stripe]
```

The mobile install/deep-link path is *not* on this picture's critical path:
the mobile SDK resolves the deep link to a code on the device and passes it
to our redemption endpoint after the user has an authenticated session
(account created). We do not try to do server-side install attribution
ourselves — that's the SDK's job.

## 4. Detailed design

### 4a. API surface

REST/JSON, served behind the existing API gateway. Versioned with a URL
prefix `/v1/...`. Why REST and not gRPC: mobile clients already speak
REST/JSON against this gateway; introducing gRPC for one feature is not
worth the toolchain spread.

| Method | Path                                  | Auth                | Purpose                                                                                |
|--------|---------------------------------------|---------------------|----------------------------------------------------------------------------------------|
| GET    | `/v1/referrals/me`                    | User JWT            | Returns the caller's referral code, share URL, and current stats (counts only).        |
| POST   | `/v1/referrals/me/code`               | User JWT            | Idempotent "ensure I have a code" — creates on first call, returns existing otherwise. |
| POST   | `/v1/referrals/redemptions`           | User JWT (referee)  | Attribute the caller (referee) to a code. Body: `{ "code": "ABC234" }`.                |
| GET    | `/v1/referrals/redemptions/me`        | User JWT            | Lets a referee see whether they're attributed and to whom (display name only).         |
| GET    | `/v1/referrals/links/resolve?c=ABC234`| Public (rate-limited) | Returns whether a code is well-formed and active. Used by the deep-link landing page.|
| GET    | `/internal/v1/referrals/stats`        | mTLS / service token | Internal aggregate endpoint for the data pipeline if CDC isn't used (see Open Q4).   |

Notes on shape:

- `GET /v1/referrals/me` is the hot read. Mobile will call it whenever the
  "Invite friends" screen opens — expect bursts. Cache server-side per user
  with a 60s TTL in Redis; cache hit serves stats from a materialized counter
  table that the activation consumer updates.
- `POST /v1/referrals/redemptions` is the critical write. It must be
  idempotent on `(referee_user_id)` — the second call with the same code
  succeeds with the existing row; the second call with a *different* code
  returns **409 Conflict** (the referee is already bound to a referrer; we
  do not silently overwrite).
- Self-redemption is rejected with **422 Unprocessable Entity** and a
  machine-readable error code.
- Errors follow RFC 7807 Problem Details (`application/problem+json`),
  consistent with the rest of the platform, with the additional fields
  `code` (stable string like `referral.self_redeem`) and `trace_id`.

### 4b. AuthN / AuthZ

- **AuthN**: existing platform JWT (HS/RS, validated at gateway *and*
  re-validated in the service via cached JWKS — defense in depth; the
  gateway has been mis-routed before).
- **AuthZ**: row-level — a user can only read their own code and stats. The
  redemption endpoint binds the *caller's* user id as the referee; the
  client cannot pass a referee id. The internal stats endpoint is mTLS or
  signed service token, never user-facing JWT.
- **Deep-link resolver** is public (it has to be — anonymous users hit it
  before account creation) but returns only well-formedness + active status,
  no PII, no referrer identity.

### 4c. Data access

Postgres 16 in our existing primary cluster. New schema `referrals`. We
explicitly choose not to introduce a new datastore — the volumes are tiny
relative to existing tables, and we want the redemption ↔ credit ledger
write to be a single transaction *or* at least a single database for
recoverability.

**Tables**

```sql
-- One row per user; created lazily on first GET /referrals/me or POST code.
CREATE TABLE referrals.codes (
    user_id        BIGINT PRIMARY KEY REFERENCES users(id),
    code           CHAR(6) NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at     TIMESTAMPTZ,
    UNIQUE (code)
);

-- One row per referee (a referee can be attributed to at most one referrer).
CREATE TABLE referrals.redemptions (
    referee_user_id   BIGINT PRIMARY KEY REFERENCES users(id),
    referrer_user_id  BIGINT NOT NULL REFERENCES users(id),
    code              CHAR(6) NOT NULL,
    source            TEXT NOT NULL CHECK (source IN ('deep_link', 'manual_entry')),
    status            TEXT NOT NULL CHECK (status IN ('pending','activated','rejected_fraud','expired')),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    activated_at      TIMESTAMPTZ,
    rejected_reason   TEXT,
    CHECK (referee_user_id <> referrer_user_id)
);
CREATE INDEX redemptions_referrer_idx ON referrals.redemptions(referrer_user_id);
CREATE INDEX redemptions_status_idx ON referrals.redemptions(status) WHERE status IN ('pending','rejected_fraud');

-- Idempotency receipts for credit issuance — guard against double-credit on retries.
CREATE TABLE referrals.credit_receipts (
    idempotency_key   TEXT PRIMARY KEY,         -- e.g. "referral:activate:{redemption_id}:referrer"
    redemption_id     BIGINT NOT NULL,           -- conceptual; pair with referee_user_id
    party             TEXT NOT NULL CHECK (party IN ('referrer','referee')),
    ledger_entry_id   TEXT NOT NULL,             -- returned by credit ledger
    amount_cents      INTEGER NOT NULL,
    issued_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Materialized counters to keep GET /referrals/me cheap.
CREATE TABLE referrals.stats (
    user_id        BIGINT PRIMARY KEY REFERENCES users(id),
    pending_count  INTEGER NOT NULL DEFAULT 0,
    activated_count INTEGER NOT NULL DEFAULT 0,
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Query budget per route**:

- `GET /v1/referrals/me`: 1 lookup on `codes` + 1 on `stats`, both PK. Or
  zero queries on a Redis cache hit. Target ≤ 2 queries.
- `POST /v1/referrals/redemptions`: 1 `INSERT ... ON CONFLICT` on
  `redemptions` + 1 conditional `UPDATE` on `stats` (referrer's
  pending_count). Single transaction. Target ≤ 3 queries.
- `GET /v1/referrals/links/resolve`: 1 lookup on `codes` (indexed by code).

**Code generation**: 6 chars from a 32-char unambiguous alphabet → ~10^9
codes. With 2.1M users today and ~2x in 18 months, collision probability on
random allocation is small but non-negligible. We generate `code` with
`secrets.token_bytes` mapped to the alphabet and rely on the `UNIQUE`
constraint to detect collisions; on conflict we retry up to 5 times. After 5
retries we widen to 7 chars (config flag) and alert. Codes are
case-insensitive on input, stored uppercase.

**Connection pool**: existing pgbouncer transaction-pooling pool;
referrals-api adds at most 50 backend connections (10 pods × 5 connections).

**Caching**: existing Redis cluster. Per-user cache for `GET /referrals/me`,
TTL 60s, invalidated on activation events.

### 4d. Latency budget

Target p95 ≤ 120 ms end-to-end for `GET /referrals/me`, p95 ≤ 200 ms for
`POST /v1/referrals/redemptions`. Mobile is a synchronous caller — UX
budget for the invite screen open is ~300 ms over cellular.

| Stage                          | p95 target | Mechanism                                              |
|--------------------------------|------------|--------------------------------------------------------|
| TLS + gateway                  | 10 ms      | shared platform gateway                                |
| JWT validation                 | 3 ms       | cached JWKS                                            |
| Redis lookup (cache hit path)  | 2 ms       | local AZ                                               |
| Postgres queries (cache miss)  | 20 ms      | PK lookups, indexed                                    |
| Serialization                  | 2 ms       | small JSON, no nested aggregation                      |
| **GET /referrals/me end-to-end** | **≤ 120 ms** | ~95% cache hit assumed once warm                    |

For the redemption write, no external calls in the synchronous path; credit
issuance is on the async activation path.

### 4e. Concurrency, rate limits, back-pressure

- Per-pod concurrency: FastAPI on `uvicorn` workers, 4 workers × 50 async
  concurrency. 10 pods → 2000 in-flight ceiling. Well above expected RPS
  (see capacity below).
- **Public deep-link resolver** is rate-limited at the gateway: 30 req/min
  per IP and 5 req/sec per code. This endpoint is the abuse magnet — a
  spammer enumerating codes hits this.
- **Redemption endpoint** is rate-limited at 1 req/sec per user JWT and
  5 req/minute per IP-device pair (defense against scripted account farms;
  see fraud below).
- Behavior at limit: HTTP 429 with `Retry-After`. No queueing.
- Graceful shutdown: 30 s drain; in-flight requests finish, new requests get
  503 with `Retry-After: 5`.

### 4f. Error contract

RFC 7807 with `code` and `trace_id`. Stable codes:

| HTTP | `code`                          | Meaning                                            |
|------|---------------------------------|----------------------------------------------------|
| 400  | `referral.invalid_format`       | Code is not 6 chars in alphabet.                   |
| 404  | `referral.code_not_found`       | Code does not exist (or is revoked).               |
| 409  | `referral.already_redeemed`     | Referee already bound to a different referrer.     |
| 422  | `referral.self_redeem`          | Tried to redeem own code.                          |
| 422  | `referral.ineligible`           | Referee account is older than X / not eligible.    |
| 429  | `referral.rate_limited`         | Rate limit; honour `Retry-After`.                  |
| 451  | `referral.fraud_blocked`        | Blocked by fraud rules (intentionally opaque).     |

`trace_id` returned in `X-Trace-Id` header on every response (success or
error) — sourced from OpenTelemetry context.

### 4g. Credit issuance — the consumer side

The activation pipeline emits `subscription.activated` events today (Open
Q3 — confirm transport: SNS→SQS, Kafka topic, or internal pubsub). We
consume them in a small worker process **co-located in the same service**
(not a separate deployment) — two entrypoints sharing the same image and
code: `web` and `worker`.

For each event:

1. Look up `redemptions` by `referee_user_id`. If none or status ≠ `pending`,
   ack the message and emit a `referrals.no_redemption` metric.
2. Run **fraud checks** (see 4h). If failed, set `status='rejected_fraud'`,
   ack, record reason, no credits.
3. Otherwise, in a single Postgres transaction:
   - `UPDATE redemptions SET status='activated', activated_at=now()`
   - `INSERT INTO credit_receipts` two rows with idempotency keys
     `referral:activate:{redemption_id}:referrer` and `:referee`.
   - `UPDATE stats` to increment `activated_count`.
4. Call the credit ledger's existing write API for each receipt. The ledger
   already supports idempotency by key, so a retry of this whole step is
   safe — the second call returns the existing ledger entry.
5. Ack the message only after step 4 succeeds.

If step 4 fails after step 3 committed: the receipts table records intent;
on next event-redelivery (or manual retry via runbook) we re-issue using
the same idempotency key. This is at-least-once → effectively-once via the
ledger's idempotency.

Consumer concurrency: low — activations are bursty but small (tens of
thousands/day at most given current MAU; see capacity). Single worker pod
is sufficient with room to scale to 3.

### 4h. Fraud guardrails (PRD calls this out, legal flagged it)

Launch posture is "make the obvious attacks expensive, accept that
sophisticated fraud survives v1 and is a follow-up".

Rules executed in order at credit-issuance time (not at redemption — we
want to *observe* attempted fraud, not blind-reject):

1. **Self-network**: same device ID, same IP /24, or matching payment
   instrument fingerprint between referrer and referee → reject.
2. **Velocity per referrer**: > 20 activations in 24 h or > 100 in 30 days
   → flag for manual review (status stays `pending`, alert fires).
3. **Velocity per IP / device**: > 5 new accounts attributed to any single
   referrer from one IP in 7 days → reject.
4. **Trial-abuse signal**: if the referee cancels and rejoins, count once;
   ledger entry uses `(referee_user_id, referrer_user_id)` as part of the
   idempotency key, not the subscription id.

What's deliberately *not* in v1: device-graph analysis, ML risk scoring,
phone-number scoring, address matching. These go in the open-questions
list with an owner.

Manual review surface: a `referrals_manual_review` table + a slack/PagerDuty
alert when count > 0. Trust/Safety team owns the queue (Open Q5 —
confirm).

### 4i. External dependencies — failure handling

| Dependency             | If down                                              | Mitigation                                                                                              |
|------------------------|------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Postgres primary       | API returns 503; mobile retries with backoff.        | Same posture as every other service. RPO bounded by existing WAL/replica setup.                          |
| Redis                  | Cache miss → DB. Slower but functional.              | Treat cache as best-effort; circuit-breaker on Redis client (`pybreaker`), 1s timeout, fallback to DB.   |
| Credit ledger API      | Activation events queue up; credits not issued yet.  | Idempotent retry on next event redelivery. Alert if backlog > 30 minutes or > 1000 events.              |
| Event bus              | We don't see activations; no credits issued.         | Backfill job (see 4j). Alert on consumer lag > 5 minutes.                                                |
| Auth (JWKS)            | All authenticated requests fail.                     | Cached JWKS with stale-while-revalidate up to 1 h.                                                       |

### 4j. Backfill / replay path

A `referrals admin backfill --since <ts>` CLI command (same service image)
that scans the existing subscription activation log for the window and
synthesizes activation events into the consumer. Used for: deploy day
catch-up, event-bus outage recovery, and the one-time historical
activation pass on launch day (Open Q6 — do we issue retroactive credits
for activations that happened *before* launch via a pre-loaded code? PM
will say no, but ask).

## 5. Alternatives considered

1. **Build referrals as its own microservice with its own DB.**
   Rejected. Volumes don't justify it, the redemption→credit-receipt write
   wants to be in one transactional boundary, and a new on-call surface
   for one feature is not warranted.
2. **Use Stripe Coupons directly (no credit ledger touch).**
   Rejected. We already have a credit ledger and reporting on it. Coupons
   would split the source of truth, complicate refunds, and make the
   marketing dashboard harder.
3. **Generate codes deterministically from user_id (e.g. hashids).**
   Rejected. Reveals user enumeration order; collisions impossible but the
   alphabet then has to be larger or longer.
4. **Server-side install attribution (we operate the deep-link tracker).**
   Rejected. The mobile SDK (AppsFlyer/Branch/Firebase) already does this
   well; reinventing is months of work and a privacy review.
5. **Synchronous credit issuance at the redemption endpoint.**
   Rejected. PRD says credits issue on *activation*, not redemption.
   Coupling redemption-time latency to the credit ledger is wrong.
6. **Do nothing / use a vendor (e.g. Friendbuy, ReferralCandy).**
   Considered. Vendors integrate at the marketing-attribution layer but
   don't own our credit ledger; we'd still build the credit-issuance side.
   The build cost here (≈ 3 eng-weeks for v1) is below the vendor
   integration + contract cost for the next 12 months. Worth revisiting
   for v2 if fraud becomes a real problem.

## 6. Non-functional requirements

Pulled from `nfrs-checklist.md`. Included where they actually matter.

### Scalability

- **Target**: Existing MAU 2.1M, +8% MoM ⇒ assume 4M MAU at 12 months.
  Even if 10% of MAU opens the invite screen weekly, that's ~570k
  `GET /referrals/me` calls/day, peak maybe 50 RPS during a campaign.
  Redemptions ≪ that — bounded by *new* user signups (today ~5–10k/day,
  expected spike to 30–50k/day during back-to-school).
- **Mechanism**: Stateless FastAPI behind ALB; HPA on CPU. Postgres is
  the only stateful piece and is well under capacity for these row
  counts (≤ 10M redemption rows projected over 3 years).
- **Scaling axis**: horizontal for the API. Vertical headroom on
  Postgres; if redemption volume 100x, we add a read replica for the
  resolver and `me` endpoints. No sharding planned or needed.
- **Verification**: load test before launch — 200 RPS sustained for 10
  min, 500 RPS burst for 1 min on `GET /referrals/me`, 50 RPS on
  redemption. Pass = p95 within targets, no error rate.

### Reliability & availability

- **Target SLO**: 99.9% monthly availability on user-facing endpoints
  (matches platform default). Tied to the existing platform's reliability
  posture — we are *not* a higher tier than billing.
- **Single points of failure**: Postgres primary. Existing failover is
  ~60 s. Acceptable.
- **Idempotency**:
  - Redemption write idempotent on PK (`referee_user_id`).
  - Credit issuance idempotent via `credit_receipts.idempotency_key` +
    ledger's own idempotency.
- **Retry policy on ledger calls**: 3 attempts, exponential backoff
  100ms/400ms/1.6s with full jitter, 2 s per-call timeout, overall
  budget 6 s. After exhaustion: leave event un-acked, let the broker
  redeliver. No client-side retry storms.

### Resilience

- Cache failure → DB fallback (not page-down).
- Event bus failure → backfill job (4j).
- Credit ledger failure → events queue up, alert at 30 min lag.
- Timeouts on every external call (Postgres 2 s, Redis 200 ms, ledger
  2 s, JWKS 1 s).

### Performance

- See latency budget (4d). p95 120 ms read, 200 ms write.
- Memory per pod: ≤ 512 MiB. CPU per pod: 500m request / 1 vCPU limit.
  Standard platform sizing.

### Observability

Metrics (Prometheus):

- `referrals_http_requests_total{route, status}` — RED rate/errors.
- `referrals_http_request_duration_seconds{route}` — RED duration,
  histogram, alert on p95.
- `referrals_redemptions_total{source, outcome}` — domain metric.
- `referrals_activations_total{outcome}` — credits issued vs blocked.
- `referrals_fraud_blocks_total{rule}` — which rule fired.
- `referrals_consumer_lag_seconds` — activation consumer lag.
- `referrals_code_generation_retries_total` — code collision warning.

Logs: structured JSON, every log line carries `trace_id`, `user_id`
(hashed for analytics; full only when authorized), `route`. No raw codes
in logs at info level (they're shareable secrets in user hands).

Traces: OpenTelemetry, propagated from gateway. 10% baseline sampling,
100% on errors.

Alerts (page vs ticket):

| Signal                                                  | Page? | Threshold                                    |
|---------------------------------------------------------|-------|----------------------------------------------|
| 5xx rate on `/v1/referrals/*` > 1% for 5 min            | Yes   | SLO burn                                     |
| p95 latency `GET /referrals/me` > 300 ms for 10 min     | Ticket| Performance                                   |
| Consumer lag > 30 min                                   | Yes   | Credit issuance halted                       |
| Manual-review queue > 50                                | Ticket| Trust/Safety follow-up                       |
| Code-generation retries > 100/h                         | Ticket| Approaching collision regime                 |
| Ledger call error rate > 5% for 10 min                  | Yes   | Cross-team incident                          |

Runbook: `<link to referrals runbook — to be added>`. Items: how to
re-drive a stuck activation, how to revoke a code, how to manually
attribute (support escalations), how to drain the manual-review queue.

### Security

- **AuthN**: existing JWT, validated twice (gateway + service). Public
  resolver endpoint clearly delineated.
- **AuthZ**: row-level. Service-internal endpoint is mTLS.
- **Data classification**: referral codes are low-sensitivity (users
  share them publicly), but the *mapping* code → user is PII-adjacent.
  Treated as PII-restricted: not logged at info, not in metric labels,
  not exported to Snowflake at user-grain without anonymization.
- **Input validation**: code regex `^[A-Z2-9&&[^OIL]]{6}$` (alphabet) at
  the edge. Reject everything else with 400.
- **Threat model**: public deep-link resolver is internet-exposed; rate
  limited and returns no PII. Authenticated endpoints assume hostile
  clients (rooted devices). Service-to-service uses mTLS.
- **Supply chain**: same lockfile + image-signing posture as the rest of
  the org's Python services.

### Cost

- API + worker: ≈ 10 pods × $30/mo = ~$300/mo compute. Negligible.
- Postgres growth: ≤ 10M rows over 3 years across all referral tables;
  ≤ 10 GB; sits in existing cluster.
- Marketing credit liability: not in scope here, but flag for finance
  the design choice of $5 × 2 per activation; the *credit ledger*
  already exposes this view.

### Compliance & privacy

- GDPR: when a user requests deletion, both `codes` and `redemptions`
  rows are pseudonymized — referrer/referee ids replaced with a
  tombstone id; aggregates preserved for fraud analysis. Plug into the
  existing platform deletion pipeline (Open Q7 — confirm hook).
- Retention: redemption rows retained 7 years (matches billing); code
  rows live for user lifetime; review/fraud rows 2 years.
- No new compliance regime triggered (no health data, no payment card
  data — credits flow via the existing PCI-in-scope ledger, not us).

### Deployment & operations

- Standard platform CI/CD: rolling deploy, 2-pod minimum during
  rollout, readiness probe gates traffic.
- Migrations: 4 new tables, all additive. `flyway`/`alembic` (whichever
  the team uses) applied pre-deploy. No `ALTER TABLE` on existing
  tables → no zero-downtime hazard.
- Feature flag: `referrals.enabled` (LaunchDarkly or equivalent),
  default off. We can gate the routes at the handler entry. Mobile
  build ships dark; server flag flipped on per-percentage rollout
  (1% → 10% → 50% → 100% over a week).
- Rollback: flag off + revert mobile screen. DB rows remain (no data
  loss); credits already issued stay issued.

### Disaster recovery

- Inherits platform DR: Postgres point-in-time recovery, daily
  snapshots, RTO 2 h, RPO 5 min. Acceptable; referrals are not a
  primary revenue path.
- Multi-region: **not** implemented. Single-region (us-east-1). If the
  region is down, the whole product is down — referrals are not the
  weak link. Explicit omission.

### Maintainability

- Tests: pytest. Unit for code generation, redemption rules, fraud
  rules. Integration with a Postgres test container and a fake ledger
  client. Contract test for the activation event consumer. Load test
  in pre-prod.
- Ownership: Growth Eng for the first 12 months, then re-evaluate
  whether it folds into Billing.
- Docs: API docs auto-generated from FastAPI OpenAPI, hosted at the
  internal docs portal.

### Extensibility

- Code length and alphabet are config — moving to 7 chars is a config
  change, not a migration.
- Reward amount is config per code-cohort (future: campaign-specific
  codes). Today: single global value.
- Codes table has room for a `cohort_id` column (deferred) for future
  campaign-specific codes without a v2 API.

### Explicitly omitted

- Multi-region active-active: see DR above.
- Per-tenant bulkheads: single-tenant product, N/A.
- Detailed cost modeling at sub-$1k/month: not worth it.

## 7. Risks & failure modes

| # | Risk                                                                                       | Likelihood | Blast radius                                                  | Mitigation                                                                  | Recovery                                                       |
|---|--------------------------------------------------------------------------------------------|------------|---------------------------------------------------------------|-----------------------------------------------------------------------------|----------------------------------------------------------------|
| 1 | Coordinated fraud farm creates fake accounts to harvest credits.                            | High       | Marketing credit liability blows out; finance escalation.     | Fraud rules (4h), velocity caps, manual-review queue, daily liability alert.| Pause flag, run reversal job that voids credits (ledger supports). |
| 2 | Activation event format changes silently when activation team refactors.                    | Medium     | All credits stop being issued; users complain.                | Contract test gated in CI on activation team's repo (cross-repo CODEOWNERS).| Backfill via the admin CLI.                                    |
| 3 | Code-collision rate climbs as user base grows.                                              | Low (within 18 mo) | Retries on insert; degraded latency.                  | Metric alert at 100 retries/h; widen to 7 chars via config flag.            | Apply config flag, rolling deploy.                             |
| 4 | Deep-link resolver becomes an open enumeration oracle (anyone can probe valid codes).       | Medium     | Privacy concern; not direct revenue.                          | Returns only "is well-formed and active", no PII; rate-limited per IP.       | Tighten limits, add CAPTCHA at the gateway if abused.          |
| 5 | Race: referee redeems and activates in the same second; the consumer reads before the redemption row is committed. | Low | One missed credit pair; visible to user as "no credit". | Redemption commit is synchronous to the user response; consumer reads via primary, not replica. | Manual re-drive via admin CLI.                |
| 6 | Mobile SDK attribution lag → user goes through deep link but lands without code populated.  | Medium     | Lost attribution (user appears organic).                      | Mobile also offers manual code entry during onboarding (PRD requirement).   | N/A — accepted loss.                                           |
| 7 | Stripe credit ledger has a backlog; credits issued in our DB but not visible to user yet.   | Low        | Confused users.                                               | UI shows "credit pending" until ledger confirms; webhook from ledger flips to "available". | Catch-up on ledger recovery.                  |
| 8 | Self-redemption attempted via the same user with two devices and two accounts.              | Medium     | One pair of fake credits.                                     | Fraud rule (matching device/IP/payment instrument).                          | Manual review.                                                 |
| 9 | The August 25 launch slips because the activation event contract is not yet defined.        | Medium     | Q3 commitment.                                                | **Open Q3 below** — must be resolved in this week's staff review.          | Fall back to polling the subscriptions table on a 1-min cron (worse but workable). |

## 8. Rollout & migration

- **Phase 0 (week 1)**: schema + API skeleton behind feature flag, off
  for all users. Internal endpoints functional; contract published for
  mobile by July 1.
- **Phase 1 (week 2)**: shadow mode — the activation consumer reads
  events but takes no credit-issuance action; logs what it *would* have
  done. Validates the contract end-to-end.
- **Phase 2 (week 3)**: enable for 1% of users (referrer-side flag).
  Watch fraud metrics for 48 h.
- **Phase 3 (weeks 4–5)**: ramp 1% → 10% → 50% → 100%.
- **Launch (Aug 25)**: 100%, marketing campaign turns on the share
  prompts in-app.

Backwards compatibility: new endpoints, no existing API touched. The
only existing-system change is the activation team adding (or
confirming) the `subscription.activated` event — coordinated.

Rollback story:

- Server: flag off. All endpoints return 404 / feature disabled. No
  data loss; in-flight pending redemptions remain in DB and can be
  resumed when re-enabled.
- Mobile: the invite screen is itself feature-flagged (server-driven
  config) — turn it off without an app release.
- Data: no destructive migrations to roll back. Credits already issued
  are not clawed back automatically; if needed, the ledger supports
  reversal entries via runbook.

## 9. Observability & operations

Already enumerated in §6 (Observability NFR). Summary of the on-call
surface:

- **Pages**: SLO burn on 5xx, consumer lag > 30 min, ledger error rate
  spike. Three real pages — keep it tight.
- **Tickets**: manual-review queue depth, code-collision rate, perf
  regression on `me`.
- **Daily**: a single "referral health" Slack message — pending
  redemptions count, activations today, credits issued, fraud blocks.
- **Runbook entries** (must exist before launch):
  1. How to revoke a referrer's code.
  2. How to manually attribute a referee (support escalation).
  3. How to re-drive a stuck activation event.
  4. How to back out credits from a fraud incident.
  5. How to flip the kill switch.

## 10. Open questions

Items marked **[BLOCKER]** flip the design; we need an answer before code.
Items without are tracking only.

1. **[BLOCKER] Mobile attribution SDK.** Which one is in-app today —
   AppsFlyer, Branch, Firebase Dynamic Links, or in-house? The deep-link
   payload format and whether the SDK fires a server callback (vs only a
   client callback) materially changes how the redemption call is wired.
   *Tentative assumption*: SDK does *client-side* deep-link resolution
   and the mobile app POSTs to `/v1/referrals/redemptions` after account
   creation. If the SDK does server-side postbacks, we add a separate
   webhook receiver (different threat model).

2. **[BLOCKER] Activation event ownership.** Who owns
   `subscription.activated`? Is it already published, and on what
   transport (SNS/SQS, Kafka topic, internal pubsub)? If it doesn't
   exist, we either (a) ask the activation team to emit it (adds a
   dependency in their roadmap) or (b) poll the subscriptions table on a
   1-minute cron (works, but worse). The TDD currently assumes (a) is
   feasible by July 15.

3. **[BLOCKER] Credit ledger write API.** We assume it exists, exposes
   an idempotent `POST /credits` keyed by an external string, and accepts
   metadata. Confirm with billing eng; if missing, we either build it or
   inline a credit-writing path here (which we'd rather not — it puts us
   in the PCI conversation).

4. **Looker / Snowflake feed.** Does the data team prefer to CDC the
   `referrals` tables (Debezium / DMS) or have us emit events / expose
   an internal stats endpoint? Default plan: CDC, no app-side work.

5. **Trust/Safety ownership of the manual-review queue.** Does this team
   exist with capacity to handle a queue from this feature, or does
   Growth Eng / Support handle it for v1? Default: Growth Eng for first
   90 days, then handoff.

6. **Retroactive credits.** Do we issue credits for activations that
   happened in a small window *before* the public launch (e.g.
   beta-flagged users) or only forward-only? Default: forward-only.

7. **GDPR deletion hook.** Confirm we can register a deletion handler
   with the existing platform deletion pipeline; or do we need to be
   polled. Default: register a handler; pseudonymize on call.

8. **Code expiry.** PRD says "currently assuming no". Recommend
   keeping codes non-expiring but allowing per-code revocation by
   admin (modeled via `revoked_at`).

9. **Eligibility rules for referees.** Is there a "must be a new user
   created within X days" rule? PRD implies yes (first paid sub) but
   does not bound how long after install. Default: no time bound; any
   referee who has never had a paid sub is eligible.

10. **Reward parameters as config vs schema.** Today: single global
    `$5 × 2`. Future campaigns may want variable amounts and per-code
    cohorts. We've left a `cohort_id` slot in the schema (deferred) —
    confirm there is no campaign already in the Q4 roadmap that would
    need this in v1.
