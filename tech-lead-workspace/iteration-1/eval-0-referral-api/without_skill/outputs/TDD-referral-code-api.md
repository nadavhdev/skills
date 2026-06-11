# TDD: Referral Code API

**Author:** Tech Lead (Backend Platform)
**Reviewers:** Staff Engineering, Growth Eng, Mobile Eng, Billing, Data, Security/Legal
**Status:** Draft for staff review
**Target launch:** Aug 25 2026 (Q3 back-to-school)
**Hard external deadline:** API contract locked by **Jul 1 2026** (mobile)
**Related PRD:** `prd-referral-api.md` (Sarah Chen, 2026-06-04)

---

## 1. Problem statement and context

We need to ship a referral program for our consumer mobile app (~2.1M MAU, 8% MoM growth) by Aug 25, 2026. A user generates a personal 6-character referral code, shares it via OS share sheet (deep link or manual entry), and when the invited friend installs and reaches "activated" (first paid subscription start), **both** users get a $5 credit on their next subscription month.

Today we have:

- A user/account service (Postgres) with the existing `users` table.
- A billing system on Stripe with our own credit ledger in Postgres.
- A mobile app that already handles deep links via the mobile attribution SDK.
- A Snowflake warehouse fed by CDC from our operational Postgres; Looker on top.
- Backend in Python (FastAPI) and Go on AWS `us-east-1`.

The work cuts across: code issuance, deep-link generation/handling, attribution at signup, activation event ingestion, credit issuance into the existing ledger, fraud controls, and analytics. This TDD scopes the **backend** side of that.

### Why now / why this design

- Marketing has a fixed external date. We optimize for **shipping correctly on time** with room to harden fraud controls post-launch.
- Credit issuance touches money. The design treats credit issuance as the system-of-record event and makes it **idempotent and auditable**, even at the cost of more code.
- We accept some bounded fraud exposure at launch (see §11) in exchange for hitting the date, and add a soft-launch / cap mechanism so a fraud outbreak cannot drain the budget before we detect it.

---

## 2. Scope

### In scope (this TDD)

- A new **Referral Service** (FastAPI) owning code issuance, redemption intake, attribution state, and credit-issuance orchestration.
- Schema for `referral_codes`, `referrals`, `referral_events`.
- REST API for mobile + internal services.
- Deep-link issuance via our existing dynamic-link provider (Branch or AppsFlyer — see §16) — backend issues the link, mobile consumes it.
- Integration with Billing's credit ledger via an existing internal RPC/API.
- Integration with the Activation event stream (the existing "first paid subscription start" event already emitted by Billing).
- Fraud / abuse controls v1 (rate limits, device/IP heuristics, manual review queue, daily cap).
- Observability, runbooks, dashboards for ops.
- Data export to Snowflake via the existing CDC pipeline; marketing dashboard built by Data team (out of scope here).

### Out of scope

- Multi-tier referrals (PRD non-goal).
- B2B referrals (PRD non-goal).
- Building the marketing Looker dashboard (Data team).
- Replacing the mobile attribution SDK.
- Cross-platform web referrals — design leaves room but launch is mobile-only.
- ML-based fraud detection (post-launch).
- Promo / campaign codes beyond personal referrals (different product, separate spec).

---

## 3. Goals and non-goals (engineering)

### Engineering goals

1. Ship a correct, idempotent referral flow by Aug 25.
2. Lock the mobile-facing API contract by **Jul 1** with no breaking changes after.
3. **Never** double-credit and **never** lose a redemption due to a race or a retry.
4. Bounded fraud exposure: configurable daily credit cap; quick kill switch.
5. Operate at 10× current MAU peak with no schema change.
6. p99 < 200ms for the mobile-hot endpoints (`GET /referral/me`, `POST /referral/redeem`).

### Non-goals

- Sub-100ms latency anywhere.
- Globally distributed writes — single-region (us-east-1) is fine.
- Real-time fraud ML.

---

## 4. High-level approach

A new **Referral Service** owns the lifecycle. It is event-driven on the inbound side (Activation event from Billing) and request/response on the outbound side (mobile and internal calls).

```
                ┌────────────────────────────────────────────────────┐
                │                   Mobile App                       │
                └──────────────┬─────────────────────┬───────────────┘
                               │ HTTPS                │ deep link
                               ▼                      ▼
                       ┌────────────────┐    ┌────────────────────┐
                       │   API Gateway  │    │ Dynamic Link Svc   │
                       │ (existing)     │    │ (Branch/AppsFlyer) │
                       └───────┬────────┘    └──────────┬─────────┘
                               │                        │ webhook
                               ▼                        ▼
                       ┌────────────────────────────────────────┐
                       │           Referral Service              │
                       │           (FastAPI, Python)             │
                       │                                         │
                       │  - POST /referral/code  (idempotent)    │
                       │  - GET  /referral/me                    │
                       │  - POST /referral/redeem                │
                       │  - GET  /referral/{code}/preview        │
                       │  - POST /internal/referral/activated    │
                       └─────┬───────────────┬─────────────┬─────┘
                             │               │             │
                             ▼               ▼             ▼
                      ┌────────────┐  ┌───────────┐ ┌──────────────┐
                      │ Postgres   │  │ Kafka /   │ │ Billing      │
                      │ (RDS)      │  │  SQS      │ │ Credit API   │
                      └────────────┘  └─────┬─────┘ └──────────────┘
                                            │
                                            ▼
                                     ┌────────────┐
                                     │ Snowflake  │
                                     │ (via CDC)  │
                                     └────────────┘
```

### Core flow (happy path)

1. **Referrer opens "Invite friends"** → mobile calls `GET /referral/me`. Service returns the user's code (creating it lazily on first call) plus a pre-built deep link.
2. **Referrer shares the code** via OS share sheet (mobile handles).
3. **Invitee taps deep link OR installs and enters code manually.**
   - Deep link path: dynamic-link provider hands the install token to the app; on signup the app calls `POST /referral/redeem` with the code and the new user's session.
   - Manual path: invitee types the code during onboarding; same `POST /referral/redeem`.
4. `POST /referral/redeem` creates a `referrals` row in state `PENDING`, atomically enforcing (a) one-referrer-per-new-user and (b) self-redemption check.
5. **Activation:** Billing emits an `account.activated` event when the user's first paid subscription starts. Referral Service consumes that event (idempotently) → transitions the `referrals` row to `ACTIVATED` → issues two ledger credit entries (one for each user) via Billing's internal credit API.
6. Both credit entries are visible in the user's next invoice.

### Why this shape

- **Separate service.** The PRD touches users, billing, attribution, and analytics — putting it in any one of those services makes it everyone else's problem. A dedicated service is small enough to ship by Jul 1 and own end-to-end.
- **Single source of truth for referral state in Postgres.** We do not rely on warehouse data for any user-visible decision.
- **Activation is an event, not a poll.** Billing already emits this; we subscribe rather than polling Stripe.
- **Credit issuance is an outbound call to Billing**, not a direct ledger write. Billing owns the ledger schema; we never write to it.

---

## 5. Detailed design

### 5.1 Code generation

- Length 6, alphabet 30 chars: `A-Z` minus `O, I` plus `2-9` minus `0, 1` → 30^6 ≈ **7.29 × 10^8** combinations.
- At launch (2.1M users, 30% adoption assumption = 630k codes) collision probability per insert is ~0.09%; with retry-on-conflict this is fine.
- **Generation strategy:** generate random 6-char string in app code; insert with `UNIQUE` constraint; on conflict, retry up to 5 times; if all 5 collide, log and 500 (effectively impossible). We do **not** use a sequence — codes must look opaque and non-enumerable.
- Codes are case-insensitive on input but stored uppercase. Validation regex: `^[A-HJ-NP-Z2-9]{6}$`.
- **Lazy issuance:** code created on first `GET /referral/me` rather than for all 2.1M users at once. Avoids a one-shot bulk write and saves space for users who never look.
- One persistent code per user (PRD requirement). No rotation in v1; design leaves a `revoked_at` column for future use.

### 5.2 Redemption flow

`POST /referral/redeem` body: `{ "code": "ABCDEF" }`. Auth: user's session token (must be the **new** user). Behavior:

1. Normalize code (uppercase, strip).
2. Validate regex; on fail → 400 `INVALID_CODE_FORMAT`.
3. Look up `referral_codes` by code; on miss → 404 `CODE_NOT_FOUND`.
4. If `referral_codes.user_id == auth_user.id` → 400 `SELF_REDEMPTION`.
5. If the new user already has a `referrals` row (any state) → 409 `ALREADY_REDEEMED` (return existing referrer's display info, so the app can show "you used X's code").
6. If new user's account is older than `REDEMPTION_WINDOW_DAYS` (default 30) → 400 `WINDOW_EXPIRED`. (PRD says "days later" is fine — we cap it to bound liability.)
7. Insert into `referrals` (state `PENDING`) inside a transaction with a unique constraint on `referred_user_id`. The DB constraint, not application logic, enforces "one referrer per new user."
8. Emit `referral.pending` event to Kafka. Return 201.

### 5.3 Activation flow

A consumer in the Referral Service subscribes to `billing.account.activated`. Per event:

1. Look up `referrals` row by `referred_user_id`. If none → ack & drop (user joined organically).
2. If state is already `ACTIVATED` or `CREDITED` → ack (idempotency on retries).
3. Transition `PENDING` → `ACTIVATING` (row lock, `SELECT FOR UPDATE`).
4. Run fraud checks (§5.6). If flagged → state `HELD_FOR_REVIEW`, emit alert, ack the event. Manual ops decision.
5. Call Billing's `POST /internal/credits` twice (once for each user) with idempotency keys: `referral:{referral_id}:referrer` and `referral:{referral_id}:referred`. Billing dedupes on the key.
6. On both successes → state `CREDITED`, persist credit IDs.
7. On any failure (4xx not retryable, 5xx retryable) → state stays `ACTIVATING`, message goes back on the queue with backoff; DLQ after N retries.
8. Emit `referral.credited` event.

**Idempotency keys** are the single mechanism we rely on for "exactly-once" credit. Stripe-style idempotency is already used by Billing — we just thread our own deterministic key.

### 5.4 Deep links

We use the existing dynamic-link provider (Branch, per the mobile team's current stack — confirm in §16).

- Backend issues the link by calling the provider's REST API at code creation time and caches the URL in `referral_codes.deep_link_url`.
- The link encodes `{ "ref_code": "ABCDEF", "ref_user_id": "<uuid>" }` in deep-link metadata. The mobile SDK surfaces this on install.
- We do **not** rely on the provider for state. The provider only carries the code into the app; redemption is still our `POST /referral/redeem` call.

### 5.5 Data model

All in Postgres (RDS), same cluster/schema as the rest of growth (separate logical DB acceptable; new service can have its own connection pool either way).

```sql
-- One row per user that has ever viewed/needed a code.
CREATE TABLE referral_codes (
    user_id            UUID PRIMARY KEY REFERENCES users(id),
    code               CHAR(6) NOT NULL UNIQUE,
    deep_link_url      TEXT,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at         TIMESTAMPTZ
);
CREATE INDEX referral_codes_code_idx ON referral_codes(code);

-- One row per (referred user) — the unique constraint enforces 1:1.
CREATE TABLE referrals (
    id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    referrer_user_id         UUID NOT NULL REFERENCES users(id),
    referred_user_id         UUID NOT NULL UNIQUE REFERENCES users(id),
    code                     CHAR(6) NOT NULL,
    state                    TEXT NOT NULL CHECK (state IN
                              ('PENDING','ACTIVATING','CREDITED',
                               'HELD_FOR_REVIEW','REJECTED','EXPIRED')),
    redeemed_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    activated_at             TIMESTAMPTZ,
    credited_at              TIMESTAMPTZ,
    referrer_credit_id       TEXT,    -- Billing ledger entry id
    referred_credit_id       TEXT,
    fraud_score              SMALLINT,
    fraud_reason             TEXT,
    redeem_ip                INET,
    redeem_device_id         TEXT,
    redeem_user_agent        TEXT,
    install_attribution_id   TEXT,    -- from the dynamic-link provider
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX referrals_referrer_idx ON referrals(referrer_user_id);
CREATE INDEX referrals_state_idx ON referrals(state) WHERE state IN ('PENDING','ACTIVATING','HELD_FOR_REVIEW');

-- Audit / event log; append-only.
CREATE TABLE referral_events (
    id             BIGSERIAL PRIMARY KEY,
    referral_id    UUID REFERENCES referrals(id),
    user_id        UUID,
    event_type     TEXT NOT NULL,    -- code_issued, redeemed, activated, credited, held, rejected
    payload        JSONB,
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX referral_events_referral_idx ON referral_events(referral_id);
```

Notes:

- `referred_user_id UNIQUE` is the load-bearing invariant for "one referrer per new user." Application code does not enforce it; the DB does.
- `state` is a string for forward-compat (new states won't require an enum migration). Constraint enforces the closed set.
- We keep `redeem_ip`, `redeem_device_id`, `redeem_user_agent` because fraud heuristics need them. These are PII-adjacent — see §10.
- `referral_events` is append-only and feeds the warehouse via CDC.

### 5.6 Fraud / abuse controls (v1)

The PRD calls this out as legally flagged and unresolved. v1 controls — all configurable via feature flag / config table:

1. **Self-referral block** — DB+app enforced.
2. **One referrer per referred user** — DB-enforced.
3. **Rate limits**: max 10 redemption attempts / hour / IP, max 50 / day / device. Use existing API-gateway rate-limit plugin.
4. **Heuristics at activation time** (synchronous, fail-open with hold):
   - New user signed up < 60s after referrer touched the link → suspicious.
   - Referrer's IP == referred user's IP at signup → suspicious.
   - Referrer's device-id == referred user's device-id → strong block.
   - Same payment fingerprint (Stripe `payment_method.fingerprint`) for referrer and referred → strong block. (Billing exposes this.)
   - Referrer has >= N (default 10) referrals activated in last 24h → hold for review.
5. **Daily credit cap**: total credits/day across all referrals capped at `$DAILY_CREDIT_CAP_USD` (config, start at $5,000 ≈ 500 activations/day). Above the cap, new activations transition to `HELD_FOR_REVIEW` and ops decide.
6. **Kill switch**: feature flag `referrals.enabled` and `referrals.credit_issuance_enabled`. Flipping `credit_issuance_enabled` off keeps redemption flowing (so we don't break mobile UX) but pauses credit issuance — events queue up.
7. **Manual review queue**: `HELD_FOR_REVIEW` rows surface in a simple internal admin tool (existing internal-admin Django app gets a new page).

This is intentionally conservative; we accept false positives in v1 and have ops triage them.

### 5.7 Activation event consumption

Billing already publishes `account.activated` on its existing event stream (Kafka topic `billing.events.v1`). We add a consumer in the Referral Service.

- **At-least-once delivery** assumed → idempotent handler (state machine + idempotency keys on Billing API).
- Consumer group dedicated to referral service.
- Lag SLO: < 5 min p99 from event publish to credit issued. Alarm at > 15 min.
- DLQ: SQS, alerted at first message; ops runbook in §13.

If Billing's event isn't reliably emitted for some edge case (e.g. plan changes), the fallback is a daily reconciliation job (§13.4).

---

## 6. API contracts

All endpoints are versioned at `/v1`. Auth via existing JWT bearer for user endpoints; mTLS for `/internal`.

### 6.1 Public (mobile-facing)

**`GET /v1/referral/me`** — returns the caller's referral code, creating it if needed.

```json
200 OK
{
  "code": "ABCDEF",
  "share_url": "https://refr.app.example/r/ABCDEF",
  "credits_earned_cents": 1500,
  "successful_referrals": 3,
  "pending_referrals": 1
}
```

Errors: 401.

**`POST /v1/referral/redeem`** — invitee submits a code.

```json
Request:  { "code": "ABCDEF", "source": "deep_link" | "manual" }
201 Created
{
  "status": "pending",
  "referrer_display_name": "Sam K.",
  "credit_on_activation_cents": 500
}
```

Errors:

| Code | HTTP | When |
|---|---|---|
| `INVALID_CODE_FORMAT` | 400 | regex fail |
| `CODE_NOT_FOUND` | 404 | no such code |
| `SELF_REDEMPTION` | 400 | code belongs to caller |
| `ALREADY_REDEEMED` | 409 | caller already has a referrer |
| `WINDOW_EXPIRED` | 400 | account too old |
| `REFERRALS_DISABLED` | 503 | kill switch |
| `RATE_LIMITED` | 429 | rate limit |

**`GET /v1/referral/{code}/preview`** — unauthenticated; used by share-sheet/web-fallback to show "X invited you." Returns minimal display name + obfuscated avatar URL. Heavily rate-limited.

### 6.2 Internal

**`POST /v1/internal/referral/activated`** — only callable by Billing (mTLS). Body mirrors the Kafka event so Billing can use HTTP or queue at its discretion. Idempotent on `event_id`.

**`POST /v1/internal/referral/{id}/release`** — releases a HELD_FOR_REVIEW row (admin only).

**`POST /v1/internal/referral/{id}/reject`** — rejects (admin only).

### 6.3 Events emitted

| Topic | Event | When |
|---|---|---|
| `referral.events.v1` | `referral.code_issued` | first code created |
| `referral.events.v1` | `referral.redeemed` | redemption row created |
| `referral.events.v1` | `referral.activated` | state → ACTIVATING |
| `referral.events.v1` | `referral.credited` | both credits issued |
| `referral.events.v1` | `referral.held` | held for review |
| `referral.events.v1` | `referral.rejected` | rejected |

All events: schema-registered (Avro/JSON-Schema, per existing convention), `event_id` UUID, `occurred_at`, `referral_id`, `actor_user_id`.

### 6.4 Stability commitments to mobile

Locked by **Jul 1**: endpoint paths, request/response shapes for `GET /v1/referral/me`, `POST /v1/referral/redeem`, and the error code enum. Server-only changes (new fields tolerated, no removals) are fine after.

---

## 7. Alternatives considered

| # | Option | Why not |
|---|---|---|
| 1 | **Build into Users service** instead of a new service. | Faster initial scaffolding but cross-cuts Billing and event consumption inside a service that doesn't own either. Coupling lifecycle of growth experiments to identity is the trap we'd regret. |
| 2 | **Build into Billing service.** | Billing already large; growth-team ownership wants its own deployable. Also keeps Billing focused on ledger correctness. |
| 3 | **Sequential / human-readable codes** (e.g. `SAMK42`). | Enumerable, abuse surface. Personal codes feel nicer but increase fraud risk and bias on choice algorithm. Rejected. |
| 4 | **Generate all codes up front for all 2.1M users.** | Wastes space (1.5M users will never share), one-shot risky migration, no benefit because lookups are by code not by user. Lazy is cheaper. |
| 5 | **Poll Stripe for "activated" instead of consuming Billing's event.** | Activation already emitted by Billing; polling Stripe duplicates logic and is racier. |
| 6 | **Write directly to Billing's credit ledger from Referral Service.** | Breaks Billing's invariant of being the single writer to its ledger. We call its API instead. |
| 7 | **Use the dynamic-link provider as the source of truth for redemption.** | We'd be tying critical money paths to a vendor's reliability + data export. Better to use it only for link metadata transport. |
| 8 | **Issue credit immediately on redemption, claw back if not activated.** | Clawbacks across Stripe are messy; PRD requires "on activation" anyway. Rejected. |
| 9 | **Synchronous credit issuance during the redeem call.** | Couples redemption latency to Billing availability; activation isn't even known yet. Async via event is the right shape. |

---

## 8. Non-functional requirements

### 8.1 Capacity

Assumptions:

- 2.1M MAU, 8% MoM → ~2.7M MAU by launch.
- Adoption: 30% of users open "Invite friends" within 90 days → ~810k codes issued in first quarter. Lazy issuance amortizes this.
- Redemptions: assume 5% of new installs come via referral. New installs ~10k/day at launch → **~500 redemptions/day**, peak ~3× = 1500/day (~17 RPS sustained, peak burst maybe 50 RPS during a marketing push).
- Activations: lag behind redemptions; same order of magnitude.

Capacity targets (10× headroom over peak):

| Endpoint | Sustained | Peak | p99 |
|---|---|---|---|
| `GET /referral/me` | 200 RPS | 1k RPS | 150ms |
| `POST /referral/redeem` | 100 RPS | 500 RPS | 200ms |
| `GET /referral/{code}/preview` | 500 RPS | 2k RPS | 100ms |
| Activation consumer | 100/s | 500/s | n/a (lag SLO) |

Storage: at 10M referrals over 3 years, < 5 GB. Trivial for RDS.

### 8.2 Performance

- All hot paths are single-row Postgres lookups by indexed column. No N+1 expected.
- Redeem path has one transaction with two writes (`referrals` insert + `referral_events` append). Acceptable.
- Activation path: one row update + 2 Billing API calls. Billing's SLA is p99 < 500ms; we tolerate up to 5s before retry.

### 8.3 Availability and reliability

- **Service availability SLO**: 99.9% monthly for read endpoints, 99.5% for writes (lower because we depend on Billing API + Kafka).
- **No data loss**: durable write to Postgres before ack on redemption; durable consumer offset commit only after state machine progresses or DLQ.
- **Degradation modes:**
  - Billing API down → activations queue in Kafka; `referrals` rows sit in `ACTIVATING`; consumer retries with backoff. No user-visible impact until they look at credits.
  - Postgres primary down → service returns 503; standby failover < 60s (existing RDS Multi-AZ).
  - Kafka down → activation events back up at Billing's broker (their problem); we lag. Redemption endpoint unaffected.
  - Dynamic-link provider down → code issuance still works; `deep_link_url` populated lazily on next read or via background backfill.

### 8.4 Consistency

- Strong consistency within Postgres (single primary, single region).
- Read-your-writes for the mobile app: `POST /redeem` returns the newly created row; subsequent `GET /me` reflects it.
- Eventual consistency between Referral Service and Billing ledger (bounded by activation consumer lag, SLO < 5 min).

### 8.5 Security

- All public endpoints behind existing API Gateway with JWT auth + WAF + rate limiting.
- `/internal` endpoints require mTLS, IP-allowlisted to the Billing service.
- PII inventory: we store `redeem_ip`, `redeem_device_id`, `redeem_user_agent`. These are classified PII-adjacent; encrypted at rest (RDS default), excluded from non-prod data dumps, deleted on GDPR/CCPA erasure together with the user row (CASCADE or service-level erasure pipeline — confirm with privacy team).
- No secrets in code; Vault for Billing API tokens and provider keys.
- Audit log: every state transition writes to `referral_events`; admin actions additionally write to the central audit log.

### 8.6 Observability

Metrics (Datadog, existing stack):

- Counter: `referral.code.issued`, `referral.redeem.{success,error_<code>}`, `referral.activated`, `referral.credited`, `referral.held`, `referral.rejected`.
- Histogram: `referral.redeem.latency_ms`, `referral.activation.processing_ms`, `referral.activation.consumer_lag_ms`.
- Gauge: `referral.daily_credit_cap.used_cents`, `referral.held_queue.depth`.

Logs: structured JSON, request-id propagated, no raw IP in INFO logs (DEBUG only, sampled).

Traces: OTel from API Gateway through service and into Billing call.

Dashboards: one ops dashboard, one growth dashboard.

Alerts (PagerDuty):

| Alert | Condition | Severity |
|---|---|---|
| Activation consumer lag | p99 > 15 min for 10 min | P2 |
| Credit issuance error rate | > 1% for 10 min | P1 |
| Held queue depth | > 100 | P3 |
| Daily cap reached | used >= cap | P3 (informational, not paging) |
| Service 5xx | > 1% for 5 min | P2 |
| DLQ has messages | depth > 0 | P2 |

### 8.7 Compliance / legal

- PRD flags fraud as a legal concern. v1 controls in §5.6 + audit trail in `referral_events` give Legal a paper trail. We schedule a v1.1 with ML-based detection.
- Terms-of-Service update needed (Legal): self-referral prohibition, multi-account farming prohibition, right to revoke credit.
- Credits are stored in our existing credit ledger; no tax/accounting changes — credits already handled.

---

## 9. Risks and failure modes

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Fraud farming drains credit budget before detection. | M | H | Daily cap (§5.6), kill switch, dashboard, ops alert at 50% cap. Accept some loss at launch. |
| R2 | Billing API outage causes pile-up of pending activations. | M | M | Idempotent retries, DLQ, backfill job. User sees delayed credit at worst. |
| R3 | Double credit due to a bug in idempotency keys. | L | H | Idempotency keys are deterministic (`referral:{id}:{role}`); Billing dedupes server-side. Add integration test that re-replays the same event 10x. |
| R4 | Mobile and backend disagree on the API after Jul 1. | M | H | Contract testing (Pact or similar) starting Jul 1; CI gate on backend PRs. |
| R5 | Dynamic-link provider misattributes installs. | M | L | We don't trust the provider for state; only for transporting the code. Manual-entry path is a fallback. |
| R6 | "Activated" event fires twice for the same user (e.g. churn-and-return). | L | M | Idempotent on `referrals.referred_user_id` UNIQUE + state machine. Only the first activation counts. |
| R7 | Hot user with 100k referrals causes pathological queries. | L | M | All queries indexed; `successful_referrals` aggregation cached or computed off CDC for top users (post-launch optimization). |
| R8 | GDPR erasure leaves orphan referrals. | L | L | Erasure pipeline nulls `referred_user_id` if the user is deleted but preserves the audit row (legal-hold compatible). Confirm with privacy. |
| R9 | Code collision storm if RNG seeded badly. | L | L | Use `secrets.token_urlsafe`-style CSPRNG; alphabet has ~7e8 codes vs 1M issued. Retry-on-conflict caps blast radius. |
| R10 | Timezone bug in daily cap (UTC vs local). | M | L | Define cap window as UTC; document in runbook; chart in growth dashboard uses same window. |

---

## 10. Privacy / data handling

- We store: `redeem_ip`, `redeem_device_id`, `redeem_user_agent`. Classified as PII-adjacent. Encrypted at rest.
- Retention: hot data indefinitely (we need it to investigate fraud disputes); audit `referral_events` 7 years.
- GDPR/CCPA erasure: existing user-erasure pipeline must include the new tables; PII columns nulled, audit row preserved.
- No PII enters logs at INFO level; debug logging samples and redacts IPs.
- Snowflake replication strips IP and device-id (CDC transform); marketing dashboard uses aggregates only.

---

## 11. Rollout / migration plan

No data migration (greenfield). Code deploy + flag rollout:

1. **W1 (Jun 9–13)**: schema in place behind feature flag; service deployed but no public route; Billing event consumer disabled.
2. **W2 (Jun 16–20)**: lock API contract with mobile (target: **Jul 1, with margin**); contract tests live.
3. **W3–W4 (Jun 23 – Jul 4)**: build out endpoints, fraud heuristics, admin tooling.
4. **W5–W6 (Jul 7–18)**: end-to-end testing in staging; load tests at 10× peak; chaos test with Billing API down.
5. **W7 (Jul 21–25)**: internal dogfood — employees only via JWT claim allowlist.
6. **W8 (Jul 28 – Aug 1)**: 1% rollout to real users (`referrals.enabled` flag, percentage-based). Watch dashboards.
7. **W9 (Aug 4–8)**: 10% → 50% if metrics clean. **Daily cap starts at $500/day at 1%, scales with rollout.**
8. **W10 (Aug 11–15)**: 100% behind kill switch, but credit issuance flag (`credit_issuance_enabled`) still off — accumulate state, verify shapes.
9. **W11 (Aug 18–22)**: enable credit issuance for the rolled-out cohort. Backfill credits for pending activations from the dogfood window if any.
10. **Aug 25**: launch announcement.

Rollback at any stage: flip `referrals.enabled` off. Credits already issued stay (no clawback in v1). Mobile is built to handle 503 `REFERRALS_DISABLED` gracefully.

---

## 12. Test plan

- **Unit**: state machine transitions, code generator (collision + alphabet), validators.
- **Integration**: full redeem → activation → credit happy path against a Billing-stub.
- **Contract tests** with mobile: Pact suite, gated in CI from Jul 1 onward.
- **Property-based** on idempotency: replay the same activation event 10x, assert exactly one credit pair.
- **Load test**: 10× peak on `GET /me` and `POST /redeem`; Locust against staging.
- **Chaos**: kill Billing API, kill Kafka, kill RDS primary (Multi-AZ failover). Assert no data loss and correct degraded behavior.
- **Fraud test harness**: scripted abuse patterns (same-device, same-IP, rapid signups) — assert hold queue catches them.
- **Manual QA**: end-to-end on real devices with real Stripe sandbox.

---

## 13. Operations / runbooks

### 13.1 Routine

- Weekly review of `HELD_FOR_REVIEW` queue by Trust & Safety.
- Daily check of cap dashboard during launch month.

### 13.2 Kill switches

- `referrals.enabled` → disables all user-facing endpoints; mobile shows "temporarily unavailable."
- `referrals.credit_issuance_enabled` → pauses credit issuance, lets state accumulate.

### 13.3 Common pages

- **Activation consumer lag**: check Kafka broker, check Billing API health, check service CPU. Likely culprits in that order.
- **Credit issuance errors**: check Billing API status; check if Billing changed idempotency-key handling; failover if regional.
- **DLQ has messages**: pull payload, classify (bug vs transient), replay or escalate.

### 13.4 Reconciliation job

A daily Go batch job (lives in existing batch infra):

- Query Billing for all activations in the last 48h.
- Cross-check against `referrals` table for any redeemed users that never moved past `PENDING`.
- Emit a metric and (optionally) re-fire the activation event into Kafka.

This is our safety net for any missed event.

---

## 14. Open questions (for staff review / cross-team)

1. **Dynamic-link provider**: Branch or AppsFlyer? Mobile team owns this decision but we need the answer for the backend integration. Default assumption: Branch.
2. **Code expiry**: PRD says no expiry. Confirm — and if so, do we still revoke on account closure? Proposal: yes, revoke on closure, no time-based expiry.
3. **Self-referral via different email/account**: legally is this still "self"? If a user creates a second account to refer themselves, our device/payment fingerprint heuristics catch it but only at activation. Acceptable?
4. **Credit issuance currency / FX**: $5 is USD. For non-US users, do we issue $5 USD-equivalent or a locale-specific amount? Need product call.
5. **What is "first paid subscription start"** exactly? Trial-to-paid conversion or any plan start? Billing team please confirm event semantics.
6. **Refund / chargeback handling**: if the new user gets refunded or churns within X days, do we claw back the referrer's credit? v1 proposal: no clawback; document for v1.1.
7. **Anonymized preview endpoint** (`GET /referral/{code}/preview`) leaks first-name + last-initial of the referrer. Privacy/Legal sign-off?
8. **Daily credit cap initial value**: $5,000/day is a guess. Marketing should set this based on campaign budget.
9. **Existing user attribution**: if a user signed up before referrals shipped and a friend then "refers" them, are they eligible? Default: no — `WINDOW_EXPIRED` for accounts > 30d old. Marketing OK with this?
10. **Tax/accounting on credits**: confirm credits don't change tax reporting — should be a no-op since the ledger already exists.
11. **Internationalization of share copy**: backend serves `share_url` only; mobile owns the share text. Confirm.
12. **Schema location**: does Referral Service get its own RDS instance or share the growth schema? Default: shared instance, separate logical schema.

---

## 15. Timeline summary

| Date | Milestone |
|---|---|
| Jun 13 | Schema + skeleton service in staging |
| **Jul 1** | **API contract locked with mobile** |
| Jul 18 | Feature-complete in staging; load + chaos tests pass |
| Jul 25 | Employee dogfood |
| Aug 1 | 1% rollout |
| Aug 15 | 100% rolled out, credit issuance still gated |
| Aug 22 | Credit issuance enabled |
| **Aug 25** | **Public launch** |

Buffer: ~10 days between feature-complete and launch is deliberate; we will use it for fraud-control hardening, not feature creep.

---

## 16. Appendix — dependencies and ownership

| Dependency | Owner | Status | Risk |
|---|---|---|---|
| Billing `account.activated` event | Billing team | Exists | Confirm semantics (Q5) |
| Billing credit-issuance API w/ idempotency keys | Billing team | Exists | Confirm `payment_method.fingerprint` exposure for fraud checks |
| Dynamic-link provider (Branch/AppsFlyer) | Mobile team | Decision pending (Q1) | Low |
| API Gateway rate-limit plugin | Platform | Exists | None |
| Kafka cluster | Platform | Exists | None |
| Snowflake CDC | Data team | Exists | None |
| Internal admin tool framework | Internal Tools | Exists | None |
| Vault for secrets | Security | Exists | None |
| Feature flag service | Platform | Exists | None |

### Team and headcount

- 2 backend engineers (Python/FastAPI) full-time, weeks 1–10.
- 0.5 SRE for staging/prod setup, observability wiring.
- 0.25 data engineer for CDC + warehouse schema sign-off.
- Coordination: 1 EM, weekly cross-team sync with Mobile, Billing, Growth, Trust & Safety.

---

*End of TDD.*
