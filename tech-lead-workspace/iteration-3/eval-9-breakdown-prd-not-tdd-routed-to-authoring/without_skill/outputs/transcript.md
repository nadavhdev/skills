# Referral Code API — Jira Ticket Breakdown

Source PRD: `/Users/nhoze/.claude/skills/tech-lead/evals/prd-referral-api.md`
Target launch: **Aug 25** (mobile contract freeze: **Jul 1**).

Before we open tickets, two flags worth raising back to Sarah/PM:

1. **Codes expiring?** PRD assumes no. I'm building for that, but it's a
   one-line change later if PM decides otherwise.
2. **Fraud.** Legal flagged "1000 fake accounts farming credits." I've
   carved a dedicated epic (E5) so it's not buried inside attribution. We
   should not ship without at least the device-fingerprint + velocity
   checks in E5-1 / E5-2.

Also: this PRD didn't go through a TDD. For most tickets below I've made
reasonable assumptions (schema shape, idempotency keys, queue vs. sync
crediting). If we want those locked down before sprint planning, a short
design doc on attribution + credit issuance would be worth half a day. I
called out the assumptions inline.

Suggested Jira hierarchy: **Initiative → Epic → Story → Sub-task**. I'm
giving you Epics + Stories; sub-tasks are for the assignee to add.

---

## Initiative: Referral Code Program (Q3 launch)

---

## Epic E1 — Referral code issuance & lookup

**Goal:** Every user has exactly one persistent, collision-free, human-friendly
code, retrievable via API.

### E1-1 — Define `referral_codes` schema and migration
- Postgres table: `user_id` (PK/FK, unique), `code` (unique, indexed),
  `created_at`.
- Code charset excludes `0 O I 1` (per PRD).
- Migration is forward-only, reviewed by DBA.
- **AC:** Migration applies cleanly on staging; rollback plan documented.
- Est: 2 pts

### E1-2 — Code generator with collision retry
- 6-char alphanumeric, excluded ambiguous chars (charset size = 32).
- Generator function with unit tests covering: charset, length,
  collision-retry path, max-retry exhaustion error.
- **AC:** 1M generated codes in test → 0 ambiguous chars, retry path
  exercised in tests.
- Est: 3 pts

### E1-3 — Issue code on user creation (backfill + new users)
- Hook into user-signup flow (FastAPI) to create the row transactionally.
- One-off backfill script for existing ~2.1M users; idempotent, batched.
- **AC:** Every existing user has a code post-backfill; new signups get one
  atomically; signup fails closed if code insert fails.
- Est: 5 pts

### E1-4 — `GET /v1/referral/me` endpoint
- Returns `{ code, share_url }` for the authenticated user.
- Auth via existing session token.
- **AC:** Endpoint documented in OpenAPI; integration test covers
  unauthenticated → 401, normal user → 200, response schema matches
  contract delivered to mobile.
- Est: 2 pts

---

## Epic E2 — Deep link + share

**Goal:** A user can share their code via the OS share sheet, and tapping the
link drops the new user into the app with the code pre-filled.

### E2-1 — Choose & configure deep-link provider
- Decision ticket: Branch.io vs. Firebase Dynamic Links vs. AppsFlyer
  (mobile-team owned, but backend needs to know the postback shape).
- **AC:** Written decision in Confluence; postback contract agreed with
  mobile + backend.
- Est: 2 pts (spike)

### E2-2 — `share_url` construction
- Given a code, produce a deep link that resolves to install + onboarding
  with code carried through.
- Server-rendered, no client logic; reused by E1-4 response.
- **AC:** Link works on iOS install-from-fresh, Android install-from-fresh,
  and app-already-installed cases (smoke-tested with mobile).
- Est: 3 pts

### E2-3 — Web fallback landing page
- For shares opened on desktop, render a minimal "Get the app" page that
  carries the code through the install via the deep-link provider.
- **AC:** Code survives App Store / Play Store round trip on a real
  device; analytics event fires on landing.
- Est: 3 pts

---

## Epic E3 — Attribution

**Goal:** When a new user installs via a code (link OR manual entry), we record
the referrer→referee link exactly once.

### E3-1 — `referrals` table & state machine
- Table: `id`, `referrer_user_id`, `referee_user_id` (nullable until
  account exists), `code`, `source` (`deeplink` | `manual`),
  `state` (`pending` | `attributed` | `activated` | `credited` | `void`),
  `created_at`, `attributed_at`, `activated_at`, `credited_at`.
- Unique constraint on `referee_user_id` (one referrer per new user).
- **Assumption flagged:** state machine — confirm in design review.
- **AC:** Migration applied; state transitions documented.
- Est: 3 pts

### E3-2 — `POST /v1/referral/redeem` (manual entry during onboarding)
- Input: `code`. Auth: new user's session.
- Rules: code must exist, referee ≠ referrer (self-redeem blocked),
  referee not already attributed (idempotent — second call with same code
  is a no-op 200, different code is 409).
- **AC:** All four rules covered by tests; metric emitted on each outcome.
- Est: 5 pts

### E3-3 — Deep-link attribution postback handler
- Receives postback from deep-link provider (E2-1) with code + new
  `user_id`; runs same attribution rules as E3-2.
- Webhook auth: HMAC signature from provider.
- **AC:** Replay-safe (same postback twice = one attribution);
  signature verification tested.
- Est: 5 pts

### E3-4 — Self-redeem + double-attribution guards
- Extract the rule checks from E3-2 / E3-3 into a single service function
  so manual + deep-link paths share one implementation.
- **AC:** Removing duplication doesn't change behavior; covered by tests
  on both entry points.
- Est: 2 pts

---

## Epic E4 — Credit issuance

**Goal:** When the referee hits "activated" (first paid subscription start),
both users get $5 credit in our ledger — exactly once.

### E4-1 — Subscribe to "subscription activated" event
- Hook into existing billing event stream (Stripe webhook → internal
  event bus). Filter to first-paid-subscription-start per user.
- **AC:** Event consumer is idempotent on Stripe event id.
- Est: 3 pts

### E4-2 — Credit issuance worker
- On activation event, look up `referrals` row in `attributed` state for
  this referee. If present: transition to `credited`, write **two** ledger
  entries ($5 referrer, $5 referee) in one DB transaction.
- Failure → leave state as `attributed`, retry from a queue with
  exponential backoff; alert after N retries.
- **Assumption flagged:** sync vs. async — proposing async via existing
  worker pool. Confirm in design.
- **AC:** Idempotent on `referral_id`; ledger sum matches expectation in
  integration test; partial failure rolls back.
- Est: 8 pts

### E4-3 — Ledger reconciliation job
- Nightly job: for every `credited` referral, assert two matching ledger
  entries exist. Discrepancy → page on-call.
- **AC:** Job runs in staging against seeded data with a deliberate
  mismatch and pages correctly.
- Est: 3 pts

---

## Epic E5 — Fraud & abuse controls

**Goal:** Make the "farm 1000 fake accounts" attack uneconomical. Legal-flagged.

### E5-1 — Per-device / per-IP velocity caps on redeem
- Reject redeem if same device id or IP has redeemed > N codes in 24h.
- Threshold owned by Trust & Safety, configurable via feature flag.
- **AC:** Synthetic load test trips the cap; metric + alert on cap-hit
  rate.
- Est: 5 pts

### E5-2 — Hold credit until activation + minimum tenure
- "Activated" requires first paid subscription start **and** the
  subscription has not refunded/charged-back within 7 days. If refund
  inside 7d → state goes to `void`, ledger entries reversed.
- **AC:** Refund-within-window path tested end to end.
- Est: 5 pts

### E5-3 — Abuse dashboard + manual void
- Internal admin endpoint: list referrals by referrer, void a referral
  (reverses ledger entries), audit logged.
- **AC:** Admin auth (existing SSO + role); audit row on every void.
- Est: 5 pts

---

## Epic E6 — Analytics & marketing dashboard

**Goal:** Marketing can see code → installs → activations daily.

### E6-1 — Emit analytics events
- Events: `referral.code_viewed`, `referral.code_shared`,
  `referral.redeem_attempted`, `referral.attributed`,
  `referral.activated`, `referral.credited`, `referral.voided`.
- Schemas registered in the existing event catalog.
- **AC:** Events visible in Snowflake landing within 1h of staging traffic.
- Est: 3 pts

### E6-2 — Snowflake model for funnel
- dbt model joining `referrals` + events → daily funnel by
  `source` (deeplink | manual) and `campaign` if present.
- **AC:** Numbers match a hand-rolled SQL on a sample day; data team
  signs off.
- Est: 5 pts (data team)

### E6-3 — Looker dashboard
- Daily view: codes generated, links shared, attributions, activations,
  credit $ issued, void rate. Owned by data team.
- **AC:** Marketing PM signs off on layout.
- Est: 3 pts (data team)

---

## Epic E7 — Launch readiness

### E7-1 — API contract published to mobile (HARD DATE Jul 1)
- OpenAPI for E1-4, E3-2, share-url shape from E2-2.
- **AC:** Mobile team has the contract checked into their repo by Jul 1.
- Est: 2 pts

### E7-2 — Feature flag + staged rollout plan
- Flag `referrals_enabled`; rollout 1% → 10% → 50% → 100% over launch week
  starting Aug 18.
- **AC:** Rollout doc reviewed by eng manager + growth PM.
- Est: 2 pts

### E7-3 — Runbook + on-call
- Runbook covers: stuck `pending` referrals, credit worker backlog,
  fraud-cap false positives, Stripe webhook outage.
- **AC:** On-call walk-through done; runbook linked from PagerDuty.
- Est: 2 pts

### E7-4 — Load test attribution + redeem paths
- Target: 10x expected back-to-school launch-day peak.
- **AC:** p95 latency budget met; no DB hotspot on `referral_codes.code`.
- Est: 3 pts

---

## Critical path to Aug 25

```
E1-1 → E1-2 → E1-3 → E1-4 ─┐
                            ├─ E7-1 (Jul 1 contract) ─ mobile work
E2-1 → E2-2 ────────────────┘
E3-1 → E3-2 / E3-3 → E3-4 ─┐
E4-1 → E4-2 → E4-3 ─────────┼─ E5-1 / E5-2 → E7-2 → E7-4 → launch
E6-1 (parallel) → E6-2 → E6-3
```

**Hard date:** E7-1 by **Jul 1** — that means E1-1 through E1-4 and E2-1/E2-2
must be done or contract-frozen by then.

---

## Recommended first sprint

E1-1, E1-2, E1-3, E1-4, E2-1, E3-1, E6-1. That gets us to a usable code
endpoint, the schema for attribution, and the deep-link decision —
unblocking the Jul 1 mobile contract.

---

## Things I'd push back on before we open these in Jira

1. **Get the fraud thresholds from Trust & Safety now**, not at launch
   week — E5-1 is sized assuming we have numbers.
2. **Stripe refund window for E5-2** (7 days is my guess) — confirm with
   billing team.
3. **Who owns the deep-link provider bill?** Branch/AppsFlyer aren't free
   at 2.1M MAU scale.
4. We should do a 1-page design doc on the attribution + credit flow
   before E3 / E4 start. Half a day, saves rework.
