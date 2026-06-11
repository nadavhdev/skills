# TDD: Vendor Inventory Sync

**Author:** Tech Lead (on behalf of Priya Subramanian / Supply Chain)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-vendor-inventory-sync.md`

---

## 0. Workload classification

This system spans more than one workload pattern, and pretending otherwise is
the most likely way to produce a bad design.

- **Dominant pattern: scheduled pull-based processing.** 11 of 12 vendors are
  pulled on a cron-like cadence (8 REST polls + 3 SFTP polls). The TDD is
  primarily structured around `references/tdd-scheduled-pull.md`.
- **One vendor is push.** Vendor K sends a daily email attachment. That path
  is effectively a (degenerate, low-volume) push receiver — handled here as a
  bounded sub-component, not a separate TDD, because it shares the
  normalisation, diff, and catalog-write pipeline with the pull path.

The pull stages (fetch, normalise, diff, apply, audit) are shared across all
12 vendors. Only the **acquisition adapter** differs per source type.

---

## 1. Problem & context

We sell ~140K SKUs across 12 vendors. Inventory is currently maintained by
hand in a Google Sheet that is ≥24h stale and frequently wrong, costing an
estimated $80K/month in "out of stock after order placed" CX incidents.

We need an automated sync that pulls inventory from each vendor, normalises
it to our SKU model, diffs against the existing product catalog, and applies
changes via the catalog service's existing REST update API. We also need to
detect when a vendor has gone silent (>24h since last good data) and an audit
trail of every inventory change with source and timestamp.

Driver: revenue loss + ops team manual toil. Soft deadline implied by
"6 weeks, 2 engineers"; shadow mode for 2 weeks before cutover.

Consumers of the sync output:
- **Product catalog service** (existing Postgres-backed service, REST update
  API, same VPC) — the only writer downstream.
- **Ops team** — consumes alerts and the audit log; arbiter during shadow
  mode.
- **Storefront team** — reads catalog (out of scope here).

---

## 2. Scope

**In scope**
- Per-vendor configuration (source type, credentials, cadence, field mapping).
- 12 connector implementations (8 REST, 3 SFTP, 1 email).
- Normalisation to internal SKU model.
- Idempotent diff + apply against the catalog service.
- Audit log of every applied change.
- Staleness detection + alerting per vendor.
- Shadow-mode run path (compute diffs, do not apply) with a reconciliation
  report.
- Operational dashboards and runbook.

**Out of scope**
- Two-way sync to vendors (explicit non-goal).
- Real-time / sub-minute inventory (explicit non-goal — "fresh enough").
- Replacing the Google Sheet on day 1 (explicit non-goal — shadow first).
- Surfacing "SKU discontinued" events to the storefront pipeline (PRD open;
  see §10).
- Onboarding new vendors beyond the 12 in scope (the design supports it as a
  variation point; rolling out vendor 13 is a separate effort).
- Cost optimisation of catalog-service writes beyond batching (catalog
  service is owned by another team).

---

## 3. High-level approach

A Django app (the team's existing stack) hosts the configuration model and
admin UI. Celery is used as the scheduler and worker runtime (Celery Beat
for cron, Celery workers for execution); Redis is the broker. AWS-hosted,
same VPC as the catalog service, `us-east-2`.

The pipeline per vendor run is:

```
[Acquisition adapter] -> [Raw payload in S3] -> [Parse + normalise]
   -> [Diff vs current catalog snapshot] -> [Apply via catalog API]
   -> [Audit log + cursor advance + freshness metric]
```

Three acquisition adapter implementations, behind a common
`InventorySource` interface:

1. **REST adapter** — async HTTP with per-vendor rate-limit token bucket;
   pagination; honours the vendor's auth (mostly API keys / OAuth2 client
   credentials).
2. **SFTP adapter** — connects via `paramiko`, lists the configured drop
   directory, downloads new files since cursor, parses CSV.
3. **Email adapter** — *not* IMAP-poll; instead, a dedicated SES inbound
   address routes Vendor K's email to S3 + SNS, which fires a Celery task
   that runs the same normalise→diff→apply pipeline. See §4c.3.

Every adapter writes the **raw payload** (HTTP body / CSV bytes / email
attachment) to S3 *before* any parsing happens. That gives us
re-processability and forensic audit independent of the parser.

Diffing uses a denormalised `catalog_snapshot` table maintained by the
sync app itself (not by querying the catalog service per SKU, which would
fan-out 140K reads). The snapshot is refreshed at the start of each
vendor's run by reading only the SKUs in scope for that vendor.

Concurrency control: one **per-vendor** distributed lock (Redis
`SET NX PX`) ensures runs for the same vendor cannot overlap. Different
vendors run in parallel.

---

## 4. Detailed design

### 4a. Schedule & trigger

Cadence per vendor (configurable per row in `Vendor` table):

| Vendor type | Default cadence | Notes |
|---|---|---|
| REST (×7)  | every 30 min | Tuned to vendor rate limits. |
| REST — Vendor F | every 1 hour | 60 req/hour total cap; see §4f. |
| SFTP (×3)  | every 1 hour | Nightly drop, but poll hourly so we discover the file ASAP and we recover quickly if the vendor re-uploads. |
| Email — Vendor K | event-driven (SES inbound) + 25-hour staleness alarm | Not actually scheduled; receives push. |

- **Scheduler:** Celery Beat with `django-celery-beat` (DB-backed schedule
  so PMs can change cadence without redeploy; audit-logged).
- **Time zone of schedule:** all schedules stored and evaluated in UTC.
  No DST handling needed because we don't store local-time schedules.
- **DST handling:** N/A given UTC. Explicitly call this out so a future
  engineer doesn't reintroduce local-tz scheduling.
- **Missed-run policy:** Beat's default is *not* to back-fire missed runs.
  We accept that; "catch up" is the next tick. For vendor K, a missed
  email is a real incident (see §4c.3 staleness alarm).

### 4b. Cursor / watermark

State stored in `vendor_sync_state` table, one row per vendor:

| column | type | purpose |
|---|---|---|
| `vendor_id` | text PK | |
| `last_success_at` | timestamptz | drives the >24h staleness alarm |
| `last_attempt_at` | timestamptz | for ops visibility |
| `last_cursor` | jsonb | adapter-specific; see below |
| `last_payload_s3_key` | text | pointer to raw payload of last successful run |
| `consecutive_failures` | int | for alerting / circuit breaker |

`last_cursor` is adapter-specific by design:

- **REST adapter** — vendor-specific. Some vendors expose `updated_since`
  (timestamp); some expose `next_page` tokens we walk to exhaustion each
  run; one (Vendor C) only supports full-catalog dump per call. The
  cursor schema is opaque JSON, the adapter interprets it.
- **SFTP adapter** — the highest filename / mtime processed. New files
  with mtime ≥ cursor are eligible. Stable filenames are an assumption
  we'll validate per vendor in onboarding (see §10).
- **Email adapter** — the SES `messageId` of the last processed mail,
  for replay protection. Plus an out-of-band store of seen message IDs
  with 7-day TTL (S3 + DynamoDB or a Postgres dedup table — see
  alternatives).

**First run** for each vendor: full-catalog ingestion. Memorialised by
running with cursor = `null`, which adapters treat as "from the
beginning". Sized below.

**Cursor lost / corrupted recovery:** the raw payload S3 bucket holds
the last N=30 days of payloads. To recover, an operator resets
`last_cursor`, optionally points the adapter at a specific S3 key (a
"replay" mode of the worker — see §8 runbook), and triggers a manual
Celery task. This is design-explicit; we will NOT add a "just rerun
from a date" CLI as part of v1 unless §10 question 4 says we should.

### 4c. Source contracts & resilience — per adapter

#### 4c.1 REST adapter (8 vendors)

- HTTP via `httpx` with explicit per-vendor timeouts (connect 5s,
  read 30s, total 90s).
- Per-vendor `token_bucket` (Redis-backed) enforcing rate limit.
  Vendor F: 60/hr ⇒ ~1 req/min sustained, so a single run cannot
  exceed 60 calls. Design implication below in §4f.
- Auth: stored in AWS Secrets Manager, one secret per vendor. Rotated
  per vendor's own policy; rotation is operator-driven, no automatic
  rotation in v1.
- Retry policy: 3 retries with exponential backoff + jitter on 5xx,
  429, and connection errors. **Not on 4xx other than 429.**
- Auth failure (401/403): **do not retry**, fail the run immediately,
  page on-call — almost always a rotated key.
- "No new data" must be distinguishable from "API down":
  - Empty 200 response ⇒ legitimate empty delta ⇒ advance
    `last_success_at`.
  - 5xx / timeout / circuit open ⇒ do NOT advance `last_success_at`;
    counts toward staleness clock.
  - 200 but malformed JSON ⇒ treat as bad data (quarantine, do not
    advance, alert).

#### 4c.2 SFTP adapter (3 vendors)

- Use `paramiko` with key-based auth where supported, password for
  the holdouts (stored in Secrets Manager).
- Each run: connect → list dir → download new files → close.
- Connection timeout 30s, total run budget 10 minutes.
- File parsing tolerates: BOM, UTF-8 vs latin-1, CRLF, leading blank
  lines, vendor-specific date formats. Header mapping is per-vendor
  config (`field_mapping`).
- A file present but unchanged across runs is detected via S3
  ETag / checksum — re-downloading the same file is a no-op
  through the diff stage anyway, but we short-circuit it for cost.
- Truncated / mid-upload files: detected by a vendor-supplied
  `.done` sentinel where the vendor supports it; otherwise we
  use mtime stability (mtime unchanged for ≥5 minutes ⇒ safe to
  read). Vendor-by-vendor in onboarding.

#### 4c.3 Email adapter (Vendor K only)

This deserves its own paragraph because the PRD names it the
weird case and it is.

- **Transport:** AWS SES inbound, dedicated address
  `inventory-k@vendors.<our-domain>`. SES receipt rule chain:
  1. Reject if the sender domain doesn't pass DMARC and isn't
     the configured allowlisted `From`.
  2. Store the raw message in S3 (`vendor-k-raw/`).
  3. Publish to SNS → SQS → Celery task.
- **Authenticity:** SPF + DKIM + DMARC enforced; `From` allowlist
  (Vendor K's known sender). IP allowlist not used (vendor uses
  cloud relays with shifting IPs). If we later see spoofing
  attempts we add S/MIME signature verification — out of scope v1.
- **Dedup:** SES `messageId` is the dedup key; persisted in a
  Postgres `email_inbox_seen` table with 7-day retention.
  Duplicates are ack-and-drop.
- **Schema tolerance:** Vendor K is human-driven; expect variation.
  We extract the *first* CSV/XLSX attachment matching `*.csv` or
  `*.xlsx`. If multiple match or none match, the message is
  routed to a `vendor-k-needs-triage/` S3 prefix with a Slack
  alert to ops. No automatic guessing.
- **Staleness:** Vendor K's contract is "daily". Alarm fires if
  no successful Vendor K ingestion in **26 hours** (24h + 2h
  grace for time-zone / sender-side delay).
- **Why not IMAP poll instead?** Two reasons. (1) IMAP requires
  storing a mailbox credential and a long-running poll that can
  miss messages around connection resets. (2) SES inbound is
  cheaper, push-based, and gives us free S3 archival. Tradeoff
  is: we have to own a subdomain MX record for `vendors.<our-domain>`
  and an SES receipt rule set — small ops cost.

### 4d. Overlap & concurrency

- **Per-vendor lock:** Redis `SET NX PX=<run-budget+grace>`.
  Default PX = 15 minutes; vendor-overridable. Two runs for the
  *same* vendor cannot overlap. Different vendors run in
  parallel — they share no per-vendor state.
- If a run is in progress and Beat fires again, the second is a
  no-op (warn-log, increment `skipped_due_to_lock_total`).
- **Lock fencing:** the lock value is a UUID per run; release
  is conditional on UUID match to avoid releasing someone else's
  lock after expiry.
- **Shared bottleneck:** all runs ultimately call the catalog
  service. We limit concurrent catalog-write workers to **4** via
  a separate Celery queue (`catalog-writes`). This protects the
  catalog service from a thundering herd if many vendors finish
  at once.

### 4e. Processing model

Per vendor run:

1. **Acquire lock** for vendor.
2. **Fetch** via adapter, **persist raw** to S3
   (`s3://inv-sync-raw/<vendor>/<yyyy>/<mm>/<dd>/<run-id>/payload.<ext>`).
3. **Parse + normalise** to a list of
   `(sku, vendor_qty, vendor_observed_at)` records.
4. **Validate** — schema, types, value ranges (qty ≥ 0, qty <
   reasonable cap e.g. 1M). Bad rows go to a per-vendor
   `quarantine` Postgres table with reason code; run does NOT
   fail unless >5% of rows are bad (configurable per vendor).
5. **Resolve SKU** — vendor SKU → internal SKU via the
   `vendor_sku_map` table. Unknown SKUs go to a separate
   `unmapped_sku` table for ops review; do not block the run.
6. **Snapshot diff** — read the current quantities for in-scope
   internal SKUs from `catalog_snapshot` (our local denormalised
   table). Compute `(sku, new_qty, old_qty)` only where
   `new_qty != old_qty`.
7. **Apply** — batched calls to catalog REST update API (batch
   size 200, configurable). Idempotency key per call =
   `sha256(vendor_id || run_id || sku)`. The catalog API must
   honour the key — see §10 q5.
8. **Audit** — write one row per applied change to
   `inventory_audit` (Postgres, partitioned by month).
9. **Advance cursor + last_success_at** in the same Postgres
   transaction as the audit batch commit ⇒ "applied & audited"
   is atomic with "cursor moved".
10. **Release lock**, emit metrics.

**Idempotency** in this design rests on three things:
- Raw payload archived before processing — replay safe.
- Diff stage compares against `catalog_snapshot`, so re-applying
  the same payload is a no-op (delta will be empty on second
  pass).
- Each catalog write carries an idempotency key derived from
  `(vendor, run, sku)`. If a write succeeds but our process
  crashes before recording it, the retry sends the same key and
  the catalog service returns the prior result.

### 4f. Backpressure / large delta / Vendor F

The hard constraint is **Vendor F: 60 req/hr total**. If Vendor F
exposes 140K SKUs / 12 vendors ≈ 12K SKUs and their API returns
~500 SKUs per page, that's 24 pages = 24 calls per full run. A
30-min cadence would burn 48 calls/hr — over budget.

Decisions:
- Vendor F runs **hourly**, not every 30 minutes.
- If Vendor F supports `updated_since`, use it; full pages only on
  first run / after a forced full-refresh.
- First-run / full-refresh of Vendor F is chunked across multiple
  hours: each run consumes ≤55 calls (5 of headroom), cursor is
  page-token-based, and the run exits when the budget is hit. The
  next run continues from the cursor.

General large-delta policy: any vendor whose run hits the
**per-run time budget** (default 10 min) exits cleanly,
checkpoints cursor, and the next tick resumes. The catalog write
side does NOT have a per-run quota; if a run finishes with 50K
diffs, all 50K get applied (250 batches of 200) before cursor
advance. We accept that one such run can take ~10 min.

### 4g. Data model (Postgres, owned by this app)

```
Vendor(
  id PK,
  name,
  source_type ENUM('rest','sftp','email'),
  source_config jsonb,           -- adapter-specific config
  poll_cron text,                -- e.g. '*/30 * * * *', null for email
  field_mapping jsonb,           -- vendor field -> internal field
  rate_limit_per_hour int,
  active bool,
  shadow_mode bool default true  -- flipped per vendor at cutover
)

VendorSkuMap(
  vendor_id, vendor_sku, internal_sku,
  PRIMARY KEY (vendor_id, vendor_sku)
)

VendorSyncState(
  vendor_id PK,
  last_success_at, last_attempt_at,
  last_cursor jsonb, last_payload_s3_key,
  consecutive_failures int
)

CatalogSnapshot(
  internal_sku PK,
  vendor_id,                     -- the vendor "owning" this SKU
  current_qty int,
  source_observed_at timestamptz,
  updated_at timestamptz
)

InventoryAudit(
  id bigserial PK,
  run_id uuid,
  vendor_id,
  internal_sku,
  old_qty, new_qty,
  source_observed_at,
  applied_at,
  applied bool,                  -- false in shadow mode
  shadow_disagreement bool       -- shadow mode: did we differ from the Google Sheet?
) PARTITION BY RANGE (applied_at)  -- monthly partitions, retain 13 months
```

Row counts to expect: ~140K snapshot rows; audit table grows by
the volume of actual changes — order of magnitude estimate
30K–100K change rows per day across all vendors after warm-up
(below).

### 4h. Capacity sizing

- **SKU count:** ~140K across 12 vendors.
- **Read volume per cycle:**
  - REST: 8 vendors × ~20 calls each (paginated, `updated_since`)
    every 30 min ⇒ 320 calls/hr.
  - SFTP: 3 vendors × 1 conn / hr.
  - Email: ~1 inbound / day.
- **Catalog write volume:** depends entirely on delta size. Steady
  state guess: 5–10% of SKUs change per day = 7K–14K writes/day,
  batched in 200s = 35–70 batch calls/day.
- **Storage:**
  - Raw payload S3: 30-day retention, ~200 MB/day estimate ⇒ ~6 GB
    rolling. Negligible.
  - `inventory_audit` Postgres: 100K rows/day × 13 months ⇒ ~40M
    rows. Easy for Postgres with monthly partitions.
- **Compute:** 2 small Celery workers (one for `default` queue,
  one for `catalog-writes`); Beat scheduler. Fits in 2 ECS Fargate
  tasks with ~1 vCPU / 2 GB each. Headroom is generous.

---

## 5. Alternatives considered

### A. Single shared cron + branching logic instead of per-vendor adapters
Have one big `sync_all_vendors` job that walks each vendor in turn.
Rejected: a slow / failing vendor blocks all others. Per-vendor
Celery tasks with per-vendor locks isolate blast radius.

### B. Pull every vendor from Airflow / Step Functions instead of Celery
More natural fit for "scheduled jobs with DAG semantics". Rejected
because (1) the team is already a Django + Celery shop with
existing on-call familiarity, (2) the workload is per-vendor
independent, not a DAG, (3) Airflow adds operational surface (a
scheduler, a metadata DB, a UI) that doesn't pay for itself at 12
vendors. Revisit if vendor count exceeds ~40 or the pipeline gains
cross-vendor stages.

### C. Webhooks from vendors
The PRD effectively forbids this: 11 of 12 vendors don't offer it,
and Vendor K is non-negotiable. We will request webhooks from any
new vendor going forward, but cannot use them as the v1 primary
mechanism. Documented as a future-state desire.

### D. IMAP poll instead of SES inbound for Vendor K
Rejected — see §4c.3 paragraph "Why not IMAP poll instead?". Boils
down to: stored mailbox credentials + missed-message risk vs a
clean S3-archived push.

### E. Bypass the catalog REST API and write directly to the catalog DB
Faster and avoids cross-service rate limiting. Rejected because
the catalog service owns its invariants (e.g. availability
thresholds, low-stock signals), and we'd duplicate that logic.
Talking through the REST API is the contract; if the catalog API
is too slow at write volume, the right fix is in the catalog
service, not bypassing it.

### F. Stream changes via Kafka / Kinesis
Rejected as overkill. The system has 12 producers, 1 consumer
(catalog), and a daily change volume in the low tens of thousands.
A managed queue / brokered stream adds cost and operational
surface with no scaling benefit at this volume. Revisit if the
catalog service grows multiple downstream readers of inventory
events.

### G. "Do nothing" (PRD push-back)
The status quo costs $80K/month and ops toil. Doing nothing is
strictly worse than any defensible v1 here. Documented for
completeness.

---

## 6. Non-functional requirements

### Reliability / freshness

- **Target:** every vendor's `last_success_at` is no older than
  `2 × poll_cadence` (e.g. 60 min for a 30-min vendor; 26h for
  Vendor K). Breach = page on-call.
- **Mechanism:** per-vendor retry with backoff; per-vendor
  staleness alarm via CloudWatch metric `vendor_staleness_minutes`.
- **Verification:** synthetic failure drill once per quarter (kill
  a vendor's credentials in staging and observe the alarm path).

### Reliability / catalog-write safety

- **Target:** zero data loss between "the sync app decided to
  apply" and "the catalog has it"; no double-application on retry.
- **Mechanism:** idempotency key per write; cursor advance is in
  the same transaction as the audit row insert; raw payload kept
  for 30 days for replay.
- **Verification:** integration test that crashes the worker
  mid-batch and verifies the next run produces no duplicates and
  no losses.

### Performance

- **Target:** p95 vendor run duration < 5 min for incremental
  runs; < 30 min for full-refresh runs.
- **Mechanism:** `updated_since`-style incremental pulls where
  available; chunked full refresh against Vendor F's rate cap.
- **Verification:** dashboard `run_duration_seconds{vendor}`,
  alert at p95 > 10 min sustained.

### Observability

- Metrics (Prometheus / CloudWatch):
  - `vendor_run_total{vendor, result=success|failed|skipped}`
  - `vendor_run_duration_seconds{vendor}`
  - `vendor_records_pulled_total{vendor}`
  - `vendor_records_changed_total{vendor}`
  - `vendor_records_quarantined_total{vendor, reason}`
  - `vendor_staleness_minutes{vendor}` (gauge)
  - `catalog_write_total{result}`
  - `catalog_write_duration_seconds`
  - `unmapped_sku_total{vendor}`
- Logs: structured JSON with `vendor_id`, `run_id`, `s3_key`.
- Traces: optional. Not required for v1 — the operations are
  sequential within a run and OpenTelemetry instrumentation can be
  added if the team adopts it broadly.
- Alerts (PagerDuty):
  - Vendor staleness breach (per vendor; see freshness SLO).
  - >3 consecutive run failures for any vendor.
  - Auth failure on any vendor (immediate).
  - Catalog write 5xx rate > 5% over 10 min.
  - SES inbound message received from a non-allowlisted sender.

### Security

- **AuthN to vendors:** credentials in AWS Secrets Manager, IAM
  scoped per-Celery-task. No credentials in env or repo.
- **AuthN from vendors (Vendor K):** SPF + DKIM + DMARC + `From`
  allowlist. Rotation if the allowlist sender changes is operator-
  driven, documented in the runbook.
- **AuthN to catalog service:** existing service-to-service auth
  (IAM-signed requests via VPC endpoint, per the catalog team's
  current contract — confirm in §10).
- **Data classification:** inventory quantities are not PII; SKU
  metadata is not PII. The audit log contains no customer data.
  This is a low-classification system.
- **Input validation:** every adapter validates types and ranges
  before write. Email attachment paths are sanitised; we never
  execute file content.
- **Threat model surface:**
  - The REST and SFTP adapters call *out*; no inbound public
    surface for those paths.
  - The SES inbound address is the only public-internet inbound;
    threat model is "anyone can send mail" — mitigated by SPF /
    DKIM / DMARC enforcement and `From` allowlist. Worst case of
    a forged successful mail: bad inventory data for Vendor K
    until ops notices. The 5%-bad-row threshold and unknown-SKU
    quarantine bound the blast radius.

### Cost

- **Estimated unit cost:** dominated by infra fixed cost
  (~$100–$200/month for 2 Fargate tasks + Redis + a small RDS
  Postgres) + S3 storage (negligible) + Secrets Manager (~$5/mo
  for 12 secrets) + SES inbound (~$0.10/1000 messages, i.e. ~$0).
- **Cost drivers:** Postgres size as `inventory_audit` grows.
  Mitigated by 13-month retention + monthly partitions.
- **Budget alert:** $500/month threshold on the AWS account tag
  for this service.

### Maintainability

- **Test strategy:** unit tests per adapter (mocked HTTP / SFTP /
  email); integration tests with `moto`/`localstack` for S3 + SES;
  contract test against catalog service using a recorded fixture;
  end-to-end "happy path" per source type in CI.
- **Documentation:** this TDD; per-vendor config docs in the repo;
  runbook (§9).
- **Ownership:** team-supply-chain-platform owns it in production.
  No second-team dependency at runtime.

### Extensibility

- New vendors plug in by inserting a `Vendor` row + a
  `VendorSkuMap` import + (if a new source type is needed) a new
  `InventorySource` subclass. The three current source types
  cover all PRD-known cases.

### Explicit NFR omissions

- **Multi-region / DR:** omitted. This is an internal data
  pipeline with no uptime SLO from external customers. If
  `us-east-2` goes down, inventory becomes stale; we revert to
  the Google Sheet temporarily. RTO/RPO is hours, which we
  accept. Backups: RDS automated daily snapshots, 7-day retention,
  is sufficient.
- **Availability SLO in the API sense:** omitted. There's no
  inbound API. The equivalent is the freshness SLO above.
- **Compliance (GDPR/HIPAA/PCI):** none apply — inventory
  quantities are not regulated data. SOC2 audit-trail
  requirements are met by the `inventory_audit` table.

---

## 7. Risks & failure modes

| # | Failure | Likelihood | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|---|
| R1 | A vendor's API silently changes schema | Medium | One vendor's data wrong | Validation + bad-row quarantine; freshness alarm fires when row threshold blocks the run | Adapter / mapping update; replay from raw S3 payload |
| R2 | A vendor's API is down for hours | High | One vendor stale | Per-vendor retry + backoff; freshness alarm | Wait for recovery; bigger delta absorbed by per-run budget over multiple runs |
| R3 | Vendor F rate-limit exceeded | Medium | Vendor F runs fail until next hour | Token bucket; per-run call cap | Automatic next-tick continuation |
| R4 | Vendor K email never arrives | Medium | Vendor K stale | 26h staleness alarm | Ops contacts vendor; manual file upload path via admin UI (planned, see §10) |
| R5 | Vendor K sends a forged email | Low | Bad data for one vendor's SKUs | SPF/DKIM/DMARC + From allowlist + quarantine threshold | Quarantine the run; replay correct one |
| R6 | Catalog API down for >30 min | Low | All vendors' applies block | Per-vendor lock unaffected; catalog-write queue drains when it returns | Queue is durable (Redis with AOF, or switch to SQS — see §10); catch up naturally |
| R7 | Catalog API non-idempotent on retry | Unknown until confirmed (§10) | Double writes possible | Idempotency key in every write; we depend on the catalog honouring it | If not honoured, we ALSO check our snapshot before write — defensive double-check |
| R8 | Cursor corruption | Low | One vendor — incorrect deltas | Cursor + audit in same txn | Operator resets cursor, replays from raw S3 |
| R9 | Database `inventory_audit` grows unbounded | Low (mitigated) | DB cost / performance | Monthly partitions, 13-month retention via partition drop | Drop oldest partition |
| R10 | Beat clock skew / overlap | Low | Wasted work | Per-vendor lock catches all overlaps | Automatic |
| R11 | Bad vendor SKU map | Medium | SKUs route to wrong internal SKU | Sample-eyeball during onboarding; `unmapped_sku` table catches the unknowns but NOT the wrong-mapped | Manual; needs ops process |
| R12 | Shadow-mode disagreement with Google Sheet | High (whole point) | Ops triage burden | Disagreement report per run; ops arbitrates (see §10 q1) | Adjust mappings; tune; cut over per vendor when stable |

---

## 8. Rollout & migration

**Phasing — 2 engineers × ~6 weeks**

| Week | Work |
|---|---|
| 1 | Skeleton: Django app, models, Celery + Beat, S3 raw bucket, secrets bootstrapping. Implement `InventorySource` base + 1 REST adapter end-to-end against Vendor A. |
| 2 | Remaining 7 REST vendors. Diff + apply pipeline. Audit table partitioning. |
| 3 | SFTP adapter + 3 SFTP vendors. Quarantine + unmapped-SKU UX. |
| 4 | SES inbound for Vendor K. Operational alerts wired. Runbook draft. |
| 5 | **Shadow mode** on for all 12 vendors. Disagreement report generated daily. Ops arbitrates. |
| 6 | Per-vendor cutover by flipping `shadow_mode=false`, starting with the highest-volume / cleanest vendors. Buffer for fix-up. |

**Shadow mode mechanics**
- `shadow_mode=true` per vendor ⇒ the pipeline runs through diff,
  writes `InventoryAudit` rows with `applied=false`, but does NOT
  call the catalog API.
- A daily report compares the shadow audit rows for the day
  against changes the ops team made via the Google Sheet that day.
  Discrepancies are flagged for ops review.
- Cutover is per vendor, not all-at-once. We can run with 5
  vendors in shadow and 7 live indefinitely.

**Rollback**
- Per-vendor: flip `shadow_mode=true` ⇒ no further catalog writes
  for that vendor. Existing writes stay (catalog has them) — no
  reversal of past quantities; ops would have to manually undo
  via the Google Sheet path if necessary. **State this in the
  runbook explicitly.**
- Whole-system: stop the Celery workers and Beat. Ops resumes the
  Google Sheet workflow. The audit table is preserved for
  forensics.
- Schema changes: forward-only migrations via Django. Each
  migration must be reversible *or* the deploy is gated.

---

## 9. Observability & operations

### Dashboards (Grafana / CloudWatch)

- Per-vendor row: cadence, last success age, last attempt result,
  records pulled / changed / quarantined, run duration p50/p95.
- System-wide: total runs/hr, catalog write rate, unmapped SKU
  trend, Redis queue depth.

### Alerts (PagerDuty)

| Alert | Threshold | Severity |
|---|---|---|
| Vendor freshness breach | `last_success_at` older than `2 × cadence` (26h for Vendor K) | P2 |
| Vendor consecutive failures | ≥3 | P3 |
| Vendor auth failure | any | P2 |
| Catalog write 5xx | >5% over 10 min | P2 |
| SES forged-sender attempt | any non-allowlisted sender to Vendor K address | P3 |
| Quarantine rate per vendor | >10% of rows in a run | P3 |
| Unmapped SKU growth | >100 new in 24h | P4 (ticket) |

### Runbook items (placeholders; to be authored in repo)

- "Vendor X is failing auth" — verify secret, rotate, replay.
- "Vendor X has stale data >Nh" — check vendor side; if vendor
  confirms healthy, check our adapter; if our adapter is healthy,
  replay last raw payload.
- "Catalog write rate is spiking" — temporarily lower
  `catalog-writes` queue concurrency.
- "Replay a specific run" — admin command:
  `python manage.py replay_run --vendor=K --run-id=...`.
- "Add a new vendor" — config + secret + SKU map + onboarding
  checklist.
- "Cut a vendor over from shadow to live" — flip flag, verify
  next 3 runs.

`<link to confluence runbook — to be added>`

---

## 10. Open questions

1. **(Design-flipping triage blocker) Catalog REST API idempotency contract.**
   Does the existing catalog update API accept an `Idempotency-Key`
   header (or equivalent) and return the prior result on
   duplicate? If **no**, we need either (a) the catalog team to add it,
   or (b) a stronger client-side guard (we read-modify-write with
   optimistic concurrency, doubling our catalog read load). **Tentative
   assumption: idempotency is supported; if not, we fall back to (b)
   and the per-vendor catalog call rate roughly doubles.** Owner:
   catalog team lead.

2. **(Design-flipping triage blocker) Shadow-mode arbitration.**
   The PRD asks "who arbitrates when shadow disagrees with the Google
   Sheet?" Tentative answer: the supply-chain ops lead, on a daily
   stand-up review of the disagreement report. We need a named human
   and a defined turnaround SLA (≤1 business day per disagreement?)
   before week 5 starts, or shadow mode produces a backlog nobody owns.

3. **(Design-flipping triage blocker) "SKU just discontinued" event surface.**
   The PRD names this as PM-open. Options: (a) the storefront team
   reads our `inventory_audit` table directly; (b) we publish an
   internal event (SNS topic / Kafka topic); (c) the storefront polls
   the catalog service. **Tentative: not in v1; we treat
   discontinuation as `qty=0` and rely on the catalog → storefront
   pipeline they already have.** Confirm with storefront team.

4. **Manual replay UX.** Do we need a per-vendor "replay last raw
   payload" admin action in v1, or is the management-command path
   (§9) enough? Recommend: command-line only in v1.

5. **Are vendor SFTP files atomic on upload?** Each of the 3 SFTP
   vendors needs to be checked: do they upload to a temp name and
   rename, or do they write in place? If in place, we need the mtime
   stability heuristic. Per-vendor confirmation needed in onboarding.

6. **Where do `VendorSkuMap` rows come from?** The PRD assumes a
   mapping exists. Recommend: import from the existing Google Sheet
   at week 1 via a one-off script, then maintain via Django admin.

7. **Catalog service rate limit for writes.** What is the
   sustainable write rate the catalog team accepts from us? Drives
   `catalog-writes` concurrency. Tentative: 50 RPS based on
   200-batch × 4 workers; confirm.

8. **Should Celery broker stay Redis, or move to SQS for the
   `catalog-writes` queue?** SQS gives at-least-once durability
   for free; Redis is simpler. Tentative: Redis is fine because
   our writes are idempotent and the queue is short-lived; revisit
   if we see message loss in shadow mode.

9. **Audit retention beyond 13 months.** The PRD asks for an audit
   log; doesn't say for how long. Tentative: 13 months for active
   query, archive older partitions to S3 (Parquet via a small
   monthly job). Confirm with compliance / data team.

10. **What's the on-call ownership during the 6-week build?** The
    PRD allocates 2 engineers but says nothing about who carries
    the pager once it's live. Need a named owning team before week
    5.
