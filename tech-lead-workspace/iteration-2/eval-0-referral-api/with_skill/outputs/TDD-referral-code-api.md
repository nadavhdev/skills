# TDD: Referral Code API

**Author:** Tech Lead (on behalf of Nadav)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-referral-api.md` (Sarah Chen, Growth PM, 2026-06-04)

> Scope note. This TDD covers the **backend Referral Code API** (sync request/response).
> The dominant workload is API serving. There is a secondary workload — a
> Stripe-event-driven credit issuance path — which we describe here as a
> sub-design rather than spinning up a second TDD. If review finds that path
> non-trivial, we will split it into its own event-driven/webhook-receiver TDD
> before build.

---

## 0. Design constraints from project conventions

No repo-level convention documents apply here. The `CLAUDE.md` in the working
directory (`/Users/nhoze/csvkit/.claude/CLAUDE.md`) governs the **csvkit CLI
suite**, which is unrelated to this feature. We are designing a new mobile-app
backend service, not a csvkit utility. The csvkit conventions are therefore
**not applicable** and are not used as constraints. If this work lands in a
backend monorepo with its own `CONTRIBUTING.md` / service template, those
conventions take precedence over this TDD's defaults and we will reconcile
before build.

Org-level constraints carried into this TDD come from the PRD itself, not from
a convention doc:

- AWS, primary region `us-east-1`.
- Backend language is **Python (FastAPI)** or **Go**. We recommend Python +
  FastAPI for this service (Section 5 explains why).
- Billing integration is **Stripe**; the credit ledger already lives in
  **Postgres** and is owned by the billing team.
- Mobile contract freeze: **July 1**. Launch: **August 25**.

---

## 1. Problem & context

The consumer mobile app (~2.1M MAU, +8% MoM) has no referral program. Growth
wants users to share a personal referral code; when an invitee installs, gets
attributed, and reaches their first paid subscription ("activated"), both the
referrer and the invitee receive **$5 of subscription credit**. Marketing
needs a per-source funnel (code → install → activation) for the back-to-school
campaign.

**Consumers of the API:**

- The mobile app (iOS / Android), authenticated as the existing user, for
  `GET my code`, `POST redeem code`, and deep-link attribution.
- An internal service-to-service call from the **billing service** (or a
  Stripe webhook hop through it) to notify "user activated" so credits can be
  issued.
- The data team / Looker, reading from Snowflake CDC of our Postgres tables
  (no direct API hit).

**Why now:** Marketing has tied the launch to the August 25 back-to-school
campaign, with a mobile contract freeze on **July 1**. Slipping Q3 means
slipping a year — the next comparable campaign window is back-to-school 2027.

---

## 2. Scope

**In scope**

- Generate and persist exactly one referral code per existing user, on demand.
- `GET /v1/referral/me` — return the caller's code + redemption stats.
- `POST /v1/referral/redeem` — invitee enters a code during onboarding.
- `POST /v1/referral/attribute` — internal endpoint called from the
  deep-link-handler on install (decoded from the deep-link parameter).
- A redemption record persisting (code, inviter\_user\_id, invitee\_user\_id,
  state, timestamps).
- An `activation` ingress (internal, from billing) that drives credit
  issuance via the existing credit ledger.
- The funnel data model used by Looker / Snowflake (table shape + CDC).
- Fraud-prevention primitives that prevent obvious self-referral and per-user
  redemption caps.

**Out of scope**

- Multi-tier referral (referrer-of-referrer credit). PRD non-goal.
- B2B / enterprise referrals. PRD non-goal.
- Cross-platform attribution beyond what the mobile SDK already provides.
  PRD non-goal.
- The Looker dashboard itself — data team owns it. We only own the source
  tables and their schema contract.
- Stripe webhook ingestion. We rely on the **existing billing service** as
  the integration point that emits "activation" to us. If billing cannot
  emit that signal, this is a blocking dependency we surface in
  Section 7 / Open Questions.
- An advanced anti-fraud / device-fingerprinting system. We ship the
  primitives (Section 4g); a real fraud engine is a follow-up.

---

## 3. High-level approach

A small, stateful **FastAPI service** ("referral-svc") fronts a dedicated
**Postgres schema** that owns `referral_codes`, `referral_redemptions`, and
`referral_activations`. Mobile clients call the service through the existing
API gateway with the user's existing JWT. On generation we deterministically
allocate a 6-char unambiguous code per user and persist it. On redemption we
write a row in `referral_redemptions` in state `pending` with a unique
constraint preventing a second redemption for the same invitee. On
activation (signal from the billing service over an internal authenticated
HTTP call, idempotency key = `invitee_user_id`), we transition the redemption
to `activated` inside a Postgres transaction and enqueue **two** credit
issuances (referrer + invitee) into the existing credit-ledger service via
its existing idempotent API. Snowflake reads via the existing Postgres → S3 →
Snowpipe CDC pipeline; no direct API path for analytics.

**Boxes-and-arrows description:**

```
Mobile app  ──JWT──▶  API Gateway  ──▶  referral-svc (FastAPI, ECS Fargate)
                                            │
                                            ├──▶  Postgres (referral schema, RDS, Multi-AZ)
                                            │       └──CDC──▶ S3 ──▶ Snowpipe ──▶ Snowflake ──▶ Looker
                                            │
                                            └──▶  credit-ledger-svc (existing)  ──▶  Stripe

Billing-svc  ──mTLS or signed internal JWT──▶  referral-svc  /v1/internal/activation
   (emits on first paid subscription start)
```

The only new long-running component is **referral-svc**. Everything else is
existing infrastructure we lean on.

---

## 4. Detailed design

### 4a. API surface

**Protocol:** REST + JSON over HTTPS. Reasons:

- Mobile clients already speak REST to every other service. Introducing
  gRPC for mobile would require an extra build step and the team has no
  prior gRPC-over-mobile production experience. The latency advantage
  doesn't matter at our QPS (Section 6).
- GraphQL would be overkill — three endpoints, no nested-resource fan-out.

**Versioning:** URL prefix `/v1/`. Breaking changes ship a `/v2/`. Old
versions are kept live for at least one mobile-app-store cycle (~90 days).

**Routes (mobile-facing, JWT-authenticated):**

| Method | Path                          | Purpose                                                 | Success | Errors                                                                                                  |
|--------|-------------------------------|---------------------------------------------------------|---------|---------------------------------------------------------------------------------------------------------|
| GET    | `/v1/referral/me`             | Return caller's code + counts (sent, pending, activated)| 200     | 401 unauthenticated                                                                                     |
| POST   | `/v1/referral/redeem`         | Invitee submits a code during onboarding                | 200     | 400 malformed code, 404 unknown code, 409 already redeemed / self-referral, 422 ineligible, 429 throttled |
| POST   | `/v1/referral/attribute`      | Deep-link install attribution (called from mobile)      | 200     | 400 malformed, 409 already attributed                                                                   |

**Routes (internal, service-to-service):**

| Method | Path                                | Purpose                                                | Auth                              |
|--------|-------------------------------------|--------------------------------------------------------|-----------------------------------|
| POST   | `/v1/internal/activation`           | Billing notifies us that `invitee_user_id` activated   | mTLS via service mesh **or** internal JWT signed by IAM-backed service identity. Choice in Open Q. |
| GET    | `/v1/internal/redemption/{id}`      | Internal lookup for ops/support                        | Same as above + audit log         |

**Schema sketches (abbreviated):**

```jsonc
// GET /v1/referral/me  → 200
{
  "code": "K7M2QH",
  "share_url": "https://app.example.com/r/K7M2QH",
  "stats": { "sent": 12, "pending": 3, "activated": 1 }
}

// POST /v1/referral/redeem  body
{ "code": "K7M2QH" }
// → 200
{ "redemption_id": "rdm_01J5G..." , "state": "pending" }
```

**Idempotency strategy on writes:**

- `POST /v1/referral/redeem`: natural idempotency on `(invitee_user_id)` —
  enforced by a unique constraint. Second call returns the same `redemption_id`
  with HTTP 200, not 409 (clients legitimately retry on flaky mobile
  networks). True conflicts (different code for same invitee) get 409.
- `POST /v1/referral/attribute`: same — natural idempotency on
  `(invitee_user_id)`.
- `POST /v1/internal/activation`: idempotency key = `invitee_user_id`. Second
  call is a no-op returning 200 with the existing state. Critical because
  the billing service will retry.

### 4b. AuthN / AuthZ

**Mobile-facing endpoints:**

- AuthN: the existing platform JWT, validated at the **API gateway** using
  cached JWKS, with **re-validation at the handler** (defense in depth — a
  misrouted edge config has historically slipped past the gateway in this
  org; cheap check, big payoff).
- AuthZ: only the JWT subject's own resources are accessible. `GET /me`
  derives the user ID from the JWT — clients cannot pass it. `redeem` writes
  against the JWT subject. There is **no** route shape where a user ID is in
  the URL on a mobile-facing endpoint, which closes the IDOR class entirely.

**Internal endpoints:**

- AuthN: **mTLS via the service mesh** is the recommendation. Falls back to a
  short-lived internal JWT signed by an IAM-backed identity if the mesh
  isn't deployed in the target environment. Choice tracked in Open Q4.
- AuthZ: restricted to known caller identities (allow-list of service SANs
  for mTLS, or `aud` claim for internal JWT). Enforced as middleware on the
  `/v1/internal/*` router, not per-handler.
- Every call to `/v1/internal/*` writes a structured audit log line
  (`actor`, `action`, `target_user_id`, `correlation_id`).

### 4c. Data access

**Database:** Postgres 15 on RDS, Multi-AZ. Same cluster as the credit
ledger to keep transactional boundaries with the ledger's existing schema.
**Schema is dedicated** (`referral`).

**Tables (sketch):**

```sql
referral.codes (
  user_id        BIGINT PRIMARY KEY,         -- one code per user
  code           CHAR(6) NOT NULL UNIQUE,    -- unambiguous alphabet, see 4f
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX codes_code_uq ON referral.codes (code);

referral.redemptions (
  id                TEXT PRIMARY KEY,         -- ULID
  code              CHAR(6) NOT NULL REFERENCES referral.codes(code),
  inviter_user_id   BIGINT NOT NULL,
  invitee_user_id   BIGINT NOT NULL UNIQUE,   -- enforces one redemption per invitee
  source            TEXT NOT NULL CHECK (source IN ('deep_link','manual_entry')),
  state             TEXT NOT NULL CHECK (state IN ('pending','activated','rejected','expired')),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  activated_at      TIMESTAMPTZ,
  CHECK (inviter_user_id <> invitee_user_id)   -- self-referral guard at DB level
);
CREATE INDEX redemptions_inviter_idx ON referral.redemptions (inviter_user_id);
CREATE INDEX redemptions_state_created_idx ON referral.redemptions (state, created_at);

referral.activations (
  redemption_id    TEXT PRIMARY KEY REFERENCES referral.redemptions(id),
  inviter_credit_ledger_txn_id  TEXT,         -- idempotency key into credit ledger
  invitee_credit_ledger_txn_id  TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Query patterns and budget (per request):**

- `GET /v1/referral/me`: 1 query for the code (or insert-if-missing), 1
  aggregate query for stats. Stats query is a `GROUP BY state` over the
  `redemptions_inviter_idx`. **Budget: 2 queries.**
- `POST /v1/referral/redeem`: 1 lookup of code → inviter, 1 insert into
  `redemptions`. Use the unique constraint on `invitee_user_id` for
  idempotency. **Budget: 2 queries.**
- `POST /v1/internal/activation`: 1 update on `redemptions` (state
  transition `pending → activated` with `WHERE state = 'pending'`), 1
  insert into `activations`, then 2 async-but-awaited HTTP calls to the
  credit ledger (one per side). All in a single Postgres transaction;
  ledger calls happen **after commit** to avoid holding a tx open across
  the network — see Section 4h.

**Caching:** None at launch. Cardinality is "per user" and the queries are
sub-millisecond. Adding Redis here adds an invalidation problem we don't
have. Revisit if `GET /me` p99 climbs past budget.

**Connection pool sizing:** PGBouncer in transaction mode in front of RDS.
Per-instance pool ~ 20 server connections; FastAPI worker concurrency tuned
so that worker × instances × 1.2 ≤ Postgres `max_connections` allocated to
the service. (Concrete numbers in Section 6.)

### 4d. Latency budget

Target: **p95 ≤ 250ms** end-to-end for mobile-facing endpoints, **p95 ≤
400ms** for `/v1/internal/activation` (includes two downstream credit-ledger
calls). Rationale: this is a "tap a button, see a result" flow, not a tight
in-app interaction; 250ms p95 is the org standard for mobile-backed REST.

| Stage                          | Target (p95) | Mechanism                                         |
|--------------------------------|--------------|---------------------------------------------------|
| TLS + gateway                  | 20ms         | shared ALB + existing API gateway                 |
| JWT validation (cached JWKS)   | 5ms          | in-process JWKS cache, refresh every 10 min       |
| Postgres queries (2)           | 15ms         | indexed PK / unique-index lookups, single AZ      |
| Application logic + serialize  | 10ms         | Pydantic models, small payload (<1KB)             |
| Network back to client         | 50ms         | shared with rest of app, mostly outside us-east-1 |
| Safety margin                  | 150ms        | absorbs cold starts, brief DB blips               |
| **End-to-end mobile read/write** | **≤ 250ms** |                                                 |

Activation path adds **2 × ~80ms** for the credit-ledger calls (existing
SLA), each with an 80ms timeout, one retry with jitter. Total budget
~400ms p95.

If the activation path is on the user's perceived onboarding latency, we
make the credit-ledger calls **fire-and-confirm-async** via an internal
`activations_pending` worker (Section 4h) and respond 202. The PRD does not
require credits to be visible the moment activation lands.

### 4e. Concurrency, rate limits, and back-pressure

**Per-instance concurrency:** FastAPI under Uvicorn workers, 4 workers per
2-vCPU Fargate task, each running asyncio. Postgres pool sized accordingly.

**Rate limits (per-tenant = per-user):**

- `GET /v1/referral/me`: 60 req/min/user (token bucket at the gateway).
- `POST /v1/referral/redeem`: **5 req/min/user** and **10 req/hour/user** —
  redemption is brute-forceable (6-char codespace = ~1.4B with our
  alphabet, but a chatty client could still try). At the limit: 429 with
  `Retry-After`.
- `POST /v1/internal/activation`: no per-user rate limit (caller is a
  trusted internal service), but **per-caller** global limit (e.g. 200
  req/s) to bound noisy-neighbor impact if billing replays a backlog.

**Brute-force defense beyond rate limit:** after **3 failed redeem attempts
per user per hour** we additionally require a (already present) on-device
CAPTCHA-ish proof token on the next redeem. This is a primitive only — not
a real fraud system (Open Q3).

**Back-pressure:** in front of Postgres we trust PGBouncer's queue. Above
that the gateway 429s. We **do not** silently queue requests in-process —
that hides saturation in p99 and breaks the connection-draining contract.

**Connection draining on deploy:** SIGTERM → stop accepting new requests
(K8s/Fargate readiness false), drain in-flight up to 30s, then SIGKILL.
Standard pattern, just need to confirm Fargate's grace-period setting
is at least 35s.

### 4f. Code generation and shape

- Alphabet: `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (32 chars; excludes
  `0/O/I/1` per PRD). 6 chars → ~1.07B codespace. At 2.1M users we use
  0.2% of the space; collision probability with random allocation is
  negligible but we still **check uniqueness on insert** via the unique
  constraint and retry on conflict.
- Generation: **CSPRNG** (`secrets.token_bytes` → base32-alphabet map),
  not a hash of `user_id` (don't want the code to leak/be derivable from
  user ID).
- Persistence: the code is generated **on first read** (lazy) inside a
  Postgres `INSERT ... ON CONFLICT DO NOTHING RETURNING code` so two
  concurrent `GET /me` calls don't double-allocate. The DB is the source
  of truth, not the generator.

### 4g. Fraud / anti-abuse primitives (launch level)

The PRD calls fraud out as a known unknown ("what if someone creates 1000
fake accounts to farm credits"). A full fraud engine is out of scope. At
launch we ship:

1. **DB-level self-referral guard:** `CHECK (inviter_user_id <>
   invitee_user_id)` on `redemptions`. Hard rejection.
2. **One redemption per invitee:** `UNIQUE (invitee_user_id)` on
   `redemptions`. Hard rejection.
3. **Activation gate = first paid subscription.** Credits only issue on
   real paid activation — abuse cost is "must pay for a sub", which is
   non-trivial.
4. **Per-IP redeem velocity** (gateway WAF rule, e.g. >50 redeems/IP/hour).
   Coarse but cheap.
5. **Daily anomaly job** (read-only, runs against Snowflake / replica)
   flagging inviters with >N activations/day or with activations clustered
   to the same device fingerprint (if mobile-SDK device ID is available
   via the deep-link payload). This is **detection, not prevention** —
   credits already issued. Sized in Open Q3.
6. **Soft hold of inviter credit** for 7 days post-activation in the
   credit ledger — gives us a reversal window if fraud is detected before
   the credit is consumed. Requires credit-ledger support; confirmed
   available per a 2025-Q4 ledger feature.

We do **not** ship: device fingerprinting at the API layer, ML-based
abuse scoring, CAPTCHA on every redeem. Those are post-launch if observed
fraud rate justifies them.

### 4h. Activation pipeline & transactional consistency

The activation path is the highest-risk operation in this design: it spans
our DB + an external service (credit ledger) and is triggered by another
service (billing) that retries. Sketch:

1. Billing → `POST /v1/internal/activation { invitee_user_id }`.
2. We `BEGIN`; `SELECT ... FOR UPDATE` on the matching `redemption` row
   (`WHERE invitee_user_id = ? AND state = 'pending'`). If no row, we
   either:
     - (a) attribution never happened — return 200 with state
       `no_redemption` (idempotent no-op),
     - (b) already activated — return 200 with state `activated`.
3. Update `state = 'activated', activated_at = now()`. Insert into
   `activations` with a generated `inviter_credit_ledger_txn_id` and
   `invitee_credit_ledger_txn_id` (ULIDs). `COMMIT`.
4. **After commit**, enqueue (in-process async task with persistent
   retry — see below) the two credit-ledger POSTs, using the
   pre-generated txn IDs as idempotency keys.

**Why pre-generate the txn IDs before the ledger calls:** so retries are
idempotent on the ledger side, and so a process crash between the two
ledger calls is recoverable from durable state (the row in `activations`
records what was *intended*; a janitor task reconciles).

**Why a janitor:** if the process dies between commit and the two HTTP
calls, the redemption is `activated` in our DB but credits aren't issued.
A reconciliation worker (runs every 5 min) scans `activations` rows whose
ledger calls have not been confirmed within 60s and replays them
(idempotent via the pre-generated txn ID). This is a small async-worker
sub-system inside the same service; if it grows, it becomes a separate
worker (Open Q2).

**Failure modes covered:**

- Ledger 5xx → retry with backoff, exponential, capped at 1 hour. After
  24h still failing, page on-call.
- Ledger 4xx (invalid request) → fail the activation, alert; this is a
  bug, not a transient.
- Our process dies mid-flow → janitor.
- Billing retries the activation → idempotency on `invitee_user_id` makes
  it safe.

### 4i. Deep-link attribution flow

The mobile app handles `https://app.example.com/r/<CODE>` via the OS
universal-link / app-link mechanism. On install, the mobile SDK passes the
code to the app on first launch. The mobile app calls
`POST /v1/referral/attribute { code }` from inside the
post-install flow, **before** onboarding starts. We:

- Look up the code → inviter\_user\_id; reject if unknown (404) or if it
  matches the current authenticated user (409 self-referral).
- Insert into `redemptions` with `source = 'deep_link'`, state `pending`.
- Return 200.

If the user enters a code **manually** during onboarding, that hits
`POST /v1/referral/redeem` instead with `source = 'manual_entry'`. The two
endpoints differ only in `source`; we could collapse them, but keeping
them separate makes the analytics funnel (`source`) trivially queryable
and lets us rate-limit manual entry more tightly than deep-link
attribution.

If a user did both — installed via deep-link and also typed a code —
the unique constraint on `invitee_user_id` means the second call returns
the existing redemption record. Mobile should treat this as success (the
code may or may not match — we accept the first one, per FR: "one new
user can only be attributed to one referrer").

### 4j. External dependencies and what happens if each is down

| Dependency           | Used for                              | If down                                                                                    |
|----------------------|---------------------------------------|--------------------------------------------------------------------------------------------|
| Postgres (RDS)       | All state                             | API hard-fails 503. Multi-AZ failover ~60s. No graceful degradation possible.              |
| Credit ledger svc    | Issuing credits on activation         | Activation path queues internally; janitor retries. Mobile-facing read/redeem unaffected.  |
| Billing service      | Source of activation signal           | Credits don't issue until billing recovers and emits backlog. Redemption flow unaffected.  |
| API gateway / JWKS   | AuthN                                 | We cache JWKS for 10 min. Beyond that, AuthN fails — same impact as the rest of the org.   |
| Snowflake / CDC      | Marketing dashboard                   | Dashboard stale; no production-path impact.                                                |

The system **gracefully degrades for marketing visibility** but **hard-fails
for the redemption write path** if Postgres is unavailable. There is no
sane way to take redemption writes when Postgres is down without inventing
an eventual-consistency story that the PRD does not ask for.

### 4k. Error contract

- Response shape: **RFC 7807 Problem Details** (`application/problem+json`)
  for all errors. The mobile platform team already parses this format from
  other services.
- Mapping:
  - `400` malformed JSON or schema-invalid body
  - `401` missing / invalid JWT
  - `403` JWT valid but not allowed (internal endpoints)
  - `404` unknown code on redeem
  - `409` conflict — self-referral, already redeemed
  - `422` semantic ineligibility (e.g. user already activated, code
    expired if expiry ever ships)
  - `429` rate-limited; includes `Retry-After`
  - `5xx` internal / dependency-down; includes correlation ID
- Every response (success and error) carries `X-Correlation-ID` derived from
  the gateway-injected trace ID. Logged on every line.

---

## 5. Alternatives considered

**A. Build on top of the existing user-profile service as a feature module
(no new service).**

- What: add tables and endpoints to the user-profile monorepo service.
- Pros: no new ECS service, fewer ops moving parts, shares the auth
  middleware natively.
- Cons: user-profile is on the critical login path; its on-call team
  resisted attaching new logic with its own DB writes and credit-ledger
  calls. Deploy cadence is gated by the user-profile team. Failure here
  could affect login.
- **Rejected** for blast-radius and ownership reasons. Login is too
  load-bearing.

**B. Greenfield Go service.**

- What: build the service in Go.
- Pros: lower per-request CPU and memory; team has Go experience too.
- Cons: the rest of the growth-adjacent backend is Python; the credit
  ledger client SDK is Python-first; Pydantic + FastAPI gives faster
  iteration for a small CRUD-ish surface; the QPS does not justify the
  language switch (Section 6).
- **Rejected.** Python + FastAPI is recommended.

**C. Event-driven activation via a queue (SQS or Kafka).**

- What: billing publishes "activation" to a topic; referral-svc consumes
  and issues credit.
- Pros: cleanly decoupled, scales independently, natural backpressure.
- Cons: adds infra we don't currently use for this team; partial-failure
  story is more complex; the activation rate is small (~thousands/day at
  launch, sized in Section 6).
- **Rejected for launch.** We use a synchronous internal POST + a small
  in-service janitor. If activation volume grows past ~50 req/s sustained,
  we revisit moving to SQS — call this out in Risks (Section 7).

**D. Store referral codes as a deterministic hash of `user_id`.**

- What: `code = base32(hmac(secret, user_id))[:6]`.
- Pros: no DB row needed to generate.
- Cons: leaks `user_id` if the secret is ever exposed; rotation is
  awkward; doesn't fit the existing alphabet without truncation/collision
  loop; gives nothing over a CSPRNG. **Rejected.**

**E. Do nothing.**

- What: skip Q3, ship later.
- Cons: misses the August 25 campaign, slips a full year on the growth
  team's roadmap, and the team has been waiting a year already.
- **Rejected** by PM (PRD constraint).

---

## 6. Non-functional requirements

NFRs are walked in the order of `references/nfrs-checklist.md`. Numbers
below assume the launch sizing: 2.1M MAU, with ~5% of MAU touching a
referral surface in a campaign month (~105k unique users in a month, peak
~50k/day, peak hourly ~10k = ~3 RPS sustained, ~30 RPS short-burst on
campaign-email send).

### Scalability

- **Capacity target.** 30 RPS p95 sustained at peak burst,
  <5 RPS steady-state. 12-month horizon: 100 RPS p95 burst (assumes the
  feature becomes part of a viral loop). Headroom design: scale to 300 RPS
  by adding Fargate tasks (Postgres can absorb 10x our query load on the
  current instance class).
- **Scaling axis.** Horizontal on the API tier. Stateful bottleneck is
  Postgres. Postgres is **not** sharded; we don't need to be at this
  scale. If we ever needed to, the natural shard key is `user_id`.
- **Stateful bottleneck.** Postgres single primary, Multi-AZ. Read
  replica available but not used at launch (queries are write-heavy or
  primary-key reads; replica lag would only complicate consistency).
- **Back-pressure.** Gateway 429 on rate-limit violations. PGBouncer
  queue absorbs short DB blips. No in-process queueing of mobile-facing
  requests.
- **Verification.** Pre-launch load test at 5× peak (150 RPS sustained,
  500 RPS burst for 5 min). Pass criterion: p95 within budget, no 5xx.

### Reliability & availability

- **Target SLO.** 99.9% monthly availability on mobile-facing endpoints
  (≈43 min downtime / month). Activation path is "best-effort within 1
  hour, 99.5% within 24h" (credits don't need to land instantly).
- **Single points of failure.** Postgres primary (mitigated by Multi-AZ,
  ~60s failover). API gateway (org-wide, not ours to mitigate). Credit
  ledger (mitigated by janitor + retry; degrades to "credits delayed").
- **Dependency failure handling.**
  - Postgres: connection retry with exponential backoff, 3 attempts,
    total budget 1.5s. Beyond that, 503.
  - Credit ledger: timeout 5s on each call, 5 retries with exponential
    backoff + jitter capped at 1 hour, surfaced via janitor.
  - Billing → us: idempotency makes their retries safe; we never reject
    a duplicate, we return current state.
- **Idempotency.** All write endpoints have a natural idempotency key
  (Section 4a). Activation idempotency is enforced at the DB row level
  via `state` transition and at the credit-ledger boundary via
  pre-generated txn IDs.
- **Verification.** Game day before launch: kill the Postgres primary,
  the credit-ledger service, and the referral-svc process mid-flight and
  measure recovery and credit-issuance correctness.

### Resilience

- **Graceful degradation.** Mobile-facing reads survive credit-ledger
  outage. Write path survives billing outage (no activations land, but
  redemptions are recorded). No reads survive Postgres outage — this is
  accepted.
- **Bulkheads.** Internal `/v1/internal/*` traffic uses a **separate
  ASG/Fargate service group** from the mobile-facing endpoints. A
  flood of activation replays cannot starve mobile reads.
- **Timeouts everywhere.** Postgres queries: 2s (overridable for the
  reconciliation worker). Credit-ledger HTTP: 5s. JWKS fetch: 2s.
  In-process locks: none — Postgres is the only locking primitive.
- **Backoff & jitter.** All retries use exponential backoff with full
  jitter (AWS-style `random(0, min(cap, base * 2^attempt))`).

### Performance

- **Latency targets.** See Section 4d. p95 ≤ 250ms mobile-facing; p95 ≤
  400ms activation; p99 ≤ 2× p95.
- **Resource budget.** Per Fargate task: 0.5 vCPU, 1 GiB RAM, 20 DB
  connections via PGBouncer. Run 4 tasks at launch (over-provisioned for
  load; sized for blast radius, not throughput).
- **N+1 / fan-out hazards.** None expected at the schema we've drawn —
  the stats query is a single `GROUP BY`. Code-reviewer call-out: never
  loop SQL queries in the redemption handler.

### Observability

- **Metrics (Prometheus / org-standard):**
  - RED per route: `request_total{route,method,status_class}`,
    `request_duration_seconds_bucket{route,...}`,
    `request_errors_total{route,error_class}`.
  - Business: `referral_redemptions_total{source,state}`,
    `referral_activations_total{outcome}`,
    `credit_ledger_call_total{side,status}`.
  - Dependency health: `postgres_query_duration_seconds_bucket`,
    `credit_ledger_call_duration_seconds_bucket`.
  - Janitor: `activation_reconciliation_lag_seconds`,
    `activation_reconciliation_retries_total{outcome}`.
- **Logs.** Structured JSON, `correlation_id`, `user_id` (when present
  and authenticated; never logged in plaintext at WARN+ without explicit
  redaction sweep). PII rules: code, user\_id are not considered PII in
  our org's data-classification doc, but inviter↔invitee linkage is a
  retention concern (Section 6 / Compliance).
- **Traces.** OpenTelemetry, propagated from the gateway. Sample rate
  10% on success, 100% on error. Span the credit-ledger call separately.
- **Alerts (SLO-based):**
  - Page: error rate on mobile-facing endpoints > 1% for 5 min;
    p95 latency > 500ms for 10 min;
    activation reconciliation lag > 30 min.
  - Ticket: code-collision retry rate > 0.1% (would indicate the codespace
    is being exhausted, which it shouldn't be); janitor backlog growing.
- **Runbook.** New runbook required at `<link to runbook — to be added>`.
  Sections: "Postgres failover", "credit-ledger outage", "billing replay
  storm", "code-collision investigation", "fraud spike playbook".

### Security

- **AuthN / AuthZ.** Section 4b.
- **Secrets.** DB credentials from AWS Secrets Manager via the existing
  IRSA pattern. Credit-ledger client uses the existing service-to-service
  credential mechanism. No new secrets introduced. Rotation cadence
  follows org policy (90d).
- **Data classification.** `referral.redemptions` links two user IDs —
  this is **PII-adjacent** because it discloses a social relationship
  between users. Retention: 24 months from `created_at`, then `inviter`
  and `invitee` IDs are pseudonymized (replaced with a one-way hash) but
  the row is preserved for fraud analytics. Encryption: RDS at rest
  (KMS), TLS in transit. Aligns with GDPR right-to-be-forgotten by
  pseudonymizing on user-deletion event from the platform identity
  service.
- **Input validation.** Pydantic at the handler boundary; code is
  regex-validated `^[A-HJ-NP-Z2-9]{6}$` (alphabet) before any DB hit.
- **Threat model.** Mobile-facing endpoints are internet-reachable
  (through gateway). Threats: brute-force code guessing (mitigated by
  rate limits + codespace size), self-referral (mitigated by DB check),
  IDOR (closed by API shape — Section 4b), replay (idempotent writes
  make replay harmless).
- **Supply chain.** Standard org pipeline; image signing via Sigstore is
  org-default. No new third-party Python deps beyond FastAPI + Pydantic
  + psycopg + the existing internal client SDKs.

### Cost

- **Estimated unit cost.** Trivial. 4 Fargate tasks × 0.5 vCPU + Multi-AZ
  RDS share — well under $500/month at launch sizing. The expensive
  thing about referrals is the **credits themselves** (issued by the
  ledger), which are a marketing-budget line, not an engineering cost.
- **Cost drivers.** RDS (already shared) and Fargate task count.
- **Cost ceiling / alert.** $1500/month service cost triggers an
  alert-to-ticket; nothing pages.

### Compliance & privacy

- **Data residency.** US-only data in `us-east-1`. No EU-specific
  treatment needed at this scale; if/when the app expands to EU, the
  referral data inherits the org-wide multi-region strategy. **N/A
  today.**
- **Retention.** 24 months on raw redemption rows, then pseudonymize.
  Audit log retained 12 months. Snowflake follows data-team retention.
- **Regulatory.** GDPR-style obligations apply to EU users via the
  org-wide deletion pipeline; we wire into it. **PCI scope: no** — we
  do not handle card data; Stripe / credit ledger does. **HIPAA: N/A.**
- **Audit log.** Every `/v1/internal/*` call; every state transition on a
  redemption.

### Deployment & operations

- **Deploy strategy.** Rolling deploy on Fargate, behind a feature flag
  `referral.api.enabled` (per-user percentage). Launch sequence: 1% → 10%
  → 50% → 100% over 2 weeks pre-campaign.
- **Rollback.** Code rollback is a flag flip. Schema is forward-only;
  all migrations are additive (new tables, new columns nullable + default).
  Rolling back a schema change is not supported by design — call out
  explicitly.
- **Feature flags.** `referral.api.enabled` (global kill switch),
  `referral.activation.enabled` (separate switch for activation pipeline,
  so we can disable credit issuance without disabling reads).
- **Config.** Service config via existing org config service. Per-env
  values: rate-limit thresholds, ledger timeout, janitor cadence.
- **Migrations.** Two migrations at launch (create schema + tables;
  create indexes CONCURRENTLY). Zero-downtime.

### Disaster recovery

- **RTO / RPO.** RTO: 1 hour (Multi-AZ failover + DNS). RPO: 5 minutes
  (RDS point-in-time recovery / continuous backups). Acceptable given
  credits can be replayed from billing on recovery.
- **Backups.** RDS automated daily snapshots, 35-day retention, weekly
  cross-region snapshot copy to `us-west-2`. Restore drill: at least
  once before launch.
- **Multi-region story.** None at launch. Single-region (`us-east-1`).
  Justification: 99.9% SLO does not require active-active; existing app
  is single-region too. Cross-region snapshots cover the doomsday case.

### Maintainability

- **Test strategy.** Unit tests for code-generation, idempotency, and
  state-machine transitions. Integration tests against a real Postgres
  in CI. Contract test for the credit-ledger client. **Load test pre-launch**
  per the Scalability section. No chaos suite at launch — manual game day
  substitutes.
- **Documentation.** This TDD. OpenAPI spec auto-generated from FastAPI
  routes, published to the internal API portal.
- **Ownership.** Growth-backend team owns this service through Q4 2026.
  Long-term owner reviewed at the Q4 planning checkpoint.

### Extensibility

- **Variation points.** New `source` values for redemption (e.g.
  `qr_code`, `email`). New activation triggers beyond "first paid
  subscription" — the activation endpoint is generic over what triggered
  it. Multi-tier referral is **not** a variation point we support; it'd
  be a redesign.
- **Versioning.** URL-versioned API (`/v1/`). Internal endpoints can
  break in lockstep with the consumer (billing) — they're not external.

---

## 7. Risks & failure modes

| # | Risk                                                                    | Likelihood | Blast radius                                              | Mitigation                                                                                  | Recovery                                                |
|---|-------------------------------------------------------------------------|------------|-----------------------------------------------------------|---------------------------------------------------------------------------------------------|---------------------------------------------------------|
| 1 | Billing service can't emit "activation" by July 1                       | Medium     | Whole launch blocks — no credits = no feature             | Confirm contract with billing team **this week**; fallback is a polling job on the ledger   | Polling fallback shippable in ~2 weeks                  |
| 2 | Fraud — fake-account farming wipes marketing budget                     | Medium     | Marketing budget burn, brand/PR if public                 | Section 4g primitives; soft credit hold; daily anomaly job; manual kill switch              | Disable activation flag; reverse soft-held credits      |
| 3 | Activation pipeline race (process dies between commit and ledger calls) | Low        | A user's credit doesn't land                              | Janitor reconciler (Section 4h)                                                             | Janitor reconciles within 5 min                         |
| 4 | Mobile contract slips past July 1                                       | Medium     | August 25 launch slips                                    | Freeze API contract by **June 24** (1-week buffer); review with mobile leads on June 17     | Mobile builds a stub against the contract while we ship |
| 5 | Code-collision retries spike (alphabet/length misconfigured)            | Low        | Slow `GET /me` first hits                                 | Pre-launch test of 1M code allocations; alert on >0.1% collision rate                       | Widen alphabet or length in a forward-compat migration  |
| 6 | Postgres failover during campaign launch                                | Low        | ~60s of 5xx                                               | Multi-AZ, pre-warmed connection pool, gateway returns 503 with `Retry-After`                | Auto-failover                                           |
| 7 | Activation backlog (billing replays a day of activations at once)       | Medium     | Activation latency spike, ledger pressure                 | Per-caller rate limit on `/internal/activation`; janitor handles overflow                   | Drain naturally within hours                            |
| 8 | Self-referral via second account on same device                         | High       | Per-instance abuse, small $                               | Detection only at launch (anomaly job + device-fingerprint via mobile SDK if available)     | Reverse credits via soft hold                           |

---

## 8. Rollout & migration

**Phasing**

- **Week -4 → -2 (pre-launch):** Service deployed to staging behind
  `referral.api.enabled=false`. Mobile integrates against staging.
- **Week -2:** Production deploy, flag off. Schema migrations applied.
  Load test in production with synthetic traffic.
- **Week -1:** Flag on at **1%** of users for 48h. Watch metrics in
  Section 9. Bump to **10%** if green.
- **Launch week (August 25):** Flag at **50%** the day before; **100%**
  on launch day morning.

**Backwards compatibility / data migration**

- N/A — greenfield service, no existing data, no existing clients.

**Rollback story**

- **Code defect:** flag flip to `false`. Mobile already handles the
  flag-off state (feature hidden).
- **Activation defect (credits issued wrongly):** flip
  `referral.activation.enabled=false`. Redemption flow stays on so
  attribution is preserved; credits don't issue. Replay activations once
  fixed (the `redemptions` rows are still there, in `pending`).
- **Schema defect:** forward-only migration to fix. No backward schema
  rollback supported by design.
- **Catastrophic abuse:** flip activation flag off, freeze ledger
  reversals via the soft-hold mechanism (Section 4g).

---

## 9. Observability & operations

(Detailed in Section 6 "Observability".) The on-call surface for this
service is small:

- **What pages a human at 3am:**
  1. Mobile-facing error rate > 1% for 5 min.
  2. p95 latency > 500ms for 10 min.
  3. Activation reconciliation lag > 30 min.
  4. Credit-ledger call error rate > 5% sustained.
- **What is a ticket:**
  1. Code-collision retry rate > 0.1%.
  2. Fraud anomaly job flags > N inviters/day.
  3. Janitor backlog growing > 1h.
- **Runbook items (must exist before launch):**
  - Postgres failover and verification.
  - Credit-ledger outage — flipping `referral.activation.enabled`.
  - Replaying a missed activation by `invitee_user_id`.
  - Manually disabling a code (soft-delete on `referral.codes`, idempotent).
  - Fraud-spike playbook.

---

## 10. Open questions

The user (Nadav) wasn't reachable for triage before drafting; the
following are the items that would normally be in the upfront triage
batch. Each carries our recommended assumption (used in the draft) so
the design isn't blocked. Decisions confirmed at the review with the
staff engineers tomorrow should be written back into the relevant
sections.

1. **Activation signal source.** Does the billing service emit
   "user activated" today, or do we need it to? **Assumption used in
   this TDD:** billing will emit it via an authenticated internal POST
   to `/v1/internal/activation` by July 1. **Fallback if not:** we
   poll the credit ledger for first paid subscription per pending
   redemption. Polling adds ~2 weeks. Decide this week. *(Blocking.)*
2. **Where the janitor / reconciler lives.** Recommended: inline in
   referral-svc, as a background asyncio task with leader election via
   Postgres advisory lock. Alternative: dedicated worker service.
   Recommend keeping it inline at launch; revisit if activation volume
   exceeds ~50 req/s sustained, at which point we move to SQS +
   dedicated worker.
3. **Fraud appetite at launch.** What rate of fraudulent credit issuance
   is acceptable in month 1? This determines whether the soft-hold
   window is 7d (recommended) vs 30d, and whether we ship device
   fingerprinting at the API layer rather than only via mobile SDK
   detection. **Assumption:** 7d hold, detection-only at launch.
4. **Service-to-service auth mechanism.** mTLS via service mesh (clean,
   stronger) vs internal JWT (works today everywhere). **Assumption:**
   internal JWT, on the basis that not every prod env has the mesh
   deployed. Confirm with platform.
5. **Code expiry.** PRD assumes "no". Our assumption: codes never
   expire. Operational concern: if a user is deleted, the code becomes
   orphaned — we soft-delete the code (`deleted_at`) so future lookups
   miss but `redemptions` history is preserved.
6. **Self-referral via second account on same device.** Do we have a
   mobile-SDK device ID available on attribution? If yes, we can add a
   detection rule cheaply. If no, this gap is accepted at launch.
7. **Activation latency UX.** Is credit expected to be visible the
   moment activation lands? **Assumption:** no — credit "within an
   hour" is acceptable, which is why the activation pipeline can be
   202+async without UX impact. Confirm with growth PM.
8. **Looker / Snowflake schema contract.** Data team owns the dashboard
   but we own the source tables. Schedule a 30-min sync with data team
   to lock the contract (column names, CDC cadence, PII handling for
   pseudonymization).
9. **Monorepo vs new repo for referral-svc.** Org default is "new repo
   per service" but the growth team's other small services live in a
   shared monorepo. Recommend monorepo for faster shared-tooling reuse.
10. **Load test fidelity.** Are we comfortable doing the pre-launch load
    test in **production** against the existing app's shared
    infrastructure (gateway, Postgres host), or do we need a staging
    environment that mirrors prod sizing? Recommend production with
    synthetic users marked via a feature flag, on a Sunday morning.
