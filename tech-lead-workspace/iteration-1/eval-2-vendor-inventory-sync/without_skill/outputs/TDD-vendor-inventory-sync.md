# TDD: Vendor Inventory Sync

**Author:** Backend Tech Lead
**Date:** 2026-06-09
**Status:** Draft for engineering review
**Related PRD:** prd-vendor-inventory-sync.md (Priya Subramanian, 2026-05-30)
**Target ship:** ~6 weeks, 2 backend engineers
**Stack:** Python 3.12, Django, Celery, Postgres, AWS (us-east-2), S3, SES

---

## 1. Problem statement

We sell ~140K SKUs across 12 vendors. Our storefront currently sees inventory
levels that are >=24h stale and routinely wrong, driving ~$80K/month in
"oversold after checkout" incidents. The fix is an automated pull-side sync
that ingests vendor inventory at a vendor-appropriate cadence and updates our
internal catalog service.

The design difficulty is **not** sync logic — it's that the 12 sources have
three fundamentally different ingestion shapes, with different failure modes,
different reliability profiles, and (in the case of Vendor K) a delivery
mechanism we don't control: a daily email attachment from a small supplier
who will not change. The system has to treat these heterogeneous sources as
first-class citizens, not paper over them.

### Source taxonomy (counted explicitly)

| Mechanism | Count | Examples we have to design for |
|---|---|---|
| REST API (polled) | 8 | Vendor F has a 60 req/hour global cap; others vary |
| SFTP nightly CSV drop | 3 | Files appear on a schedule; can be late / missing / partial |
| Email attachment | 1 | Vendor K. Human-sent, daily-ish, occasionally with the wrong file |

Everything downstream of ingestion is identical (normalize -> diff -> apply
-> audit -> alert). That asymmetry is the central design lever.

---

## 2. Goals and non-goals

### In scope (this project)

- Per-vendor configuration of source type, credentials, schedule, field map.
- Three ingestion adapters: REST poller, SFTP fetcher, email-attachment receiver.
- Common normalization to internal SKU model.
- Diff against catalog; PATCH only what changed via existing catalog REST API.
- Staleness detection: alert when any vendor has had no successful sync in >24h
  (configurable per vendor).
- Append-only audit log of every applied inventory change with source, value,
  timestamp, and the run that produced it.
- Two-week shadow mode (compute and log diffs; do not apply to catalog).

### Explicitly out of scope (per PRD; restating so the team doesn't drift)

- Two-way sync back to vendors.
- Real-time / sub-hourly freshness.
- Replacing the ops Google Sheet on day 1.
- A "discontinued SKU" event bus to the storefront team (see Open Questions).

### Success criteria (measurable)

- p95 end-to-end freshness from vendor data availability to catalog apply:
  - REST vendors: <= 15 min after a successful poll window.
  - SFTP vendors: <= 30 min after file lands on SFTP.
  - Email vendor: <= 30 min after email arrives in the receiving mailbox.
- Oversell incidents (CX metric) drop by >=70% within 60 days of full rollout.
- Stale-vendor alert fires within 1 hour of the 24h staleness threshold being
  crossed (no later than 25h after last successful sync).
- Zero silent catalog corruption: every applied change has a corresponding
  audit row; reconciliation job passes daily.

---

## 3. High-level approach

A single Django service ("inventory-sync") with Celery workers, deployed in
the existing AWS VPC alongside the catalog service. One **pipeline** per
vendor run, composed of pluggable stages:

```
  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐   ┌────────────┐   ┌────────┐
  │  Ingest      │ → │  Parse +     │ → │  Normalize  │ → │  Diff      │ → │ Apply  │
  │  (per type)  │   │  Validate    │   │  to SKU     │   │  vs catalog│   │  +Audit│
  └──────────────┘   └──────────────┘   └─────────────┘   └────────────┘   └────────┘
        │                  │                  │                  │              │
        └──── raw blob ───→ S3 (immutable, source-of-truth for replays) ────────┘
```

Key shape decisions:

1. **Three ingestion adapters, one downstream pipeline.** REST/SFTP/Email
   adapters differ only in how they produce a "raw payload object in S3 +
   a `VendorRun` row in Postgres". Everything after that is shared code.
2. **S3 is the source of record for raw vendor data.** Every inbound payload
   (JSON page, CSV file, email attachment) is persisted to
   `s3://inventory-sync-raw/<vendor>/<run_id>/...` *before* parsing. This
   makes parser bugs cheap (re-run from S3) and gives us an auditable
   chain-of-custody for the email vendor.
3. **Celery for orchestration, not Airflow / Step Functions.** The org
   already runs Celery; introducing another scheduler for ~30 scheduled
   jobs is not worth it. Celery Beat handles cadence; `celery.chord` is
   enough for the small fan-in we need.
4. **Pull-only.** Catalog service exposes a REST update API. We call it.
   No DB-level coupling; no shared schema with catalog.
5. **Shadow mode is a config flag per vendor, not a code branch.** The
   `Apply` stage checks `vendor.apply_enabled` and either calls catalog
   or just writes a `would_apply` audit row. Cutover is config-only.

---

## 4. Detailed design

### 4.1 Components

- **inventory-sync-web** — Django app. Admin UI for per-vendor config,
  manual replay triggers, run history view, dashboards.
- **inventory-sync-worker** — Celery workers running ingestion + pipeline
  tasks. At least two queues:
  - `ingest` (slow / IO-bound, sized for the SFTP and large REST pulls).
  - `pipeline` (CPU-bound parse/diff/apply).
- **inventory-sync-beat** — Celery Beat scheduler driven from the
  `VendorSchedule` table, not from static `CELERYBEAT_SCHEDULE`. We must
  be able to change a vendor's cadence without redeploy.
- **inbound-email pipeline** — SES inbound rule writes raw MIME to
  `s3://inventory-sync-inbound-email/`, which triggers an SQS message,
  which a dedicated Celery task consumes. (Details in 4.2.3.)
- **Postgres** — config, run state, audit log, dedupe ledger.
- **S3 buckets** — raw payloads (`-raw`), inbound email (`-inbound-email`).
- **CloudWatch + existing org observability (Datadog assumed)** for logs,
  metrics, alerts.

### 4.2 Ingestion adapters

All three adapters share an interface:

```python
class IngestionAdapter(Protocol):
    def run(self, vendor: Vendor, run: VendorRun) -> IngestResult:
        """Produce zero or more raw payloads in S3 and update `run`.

        On success: run.status = 'INGESTED', run.raw_s3_keys = [...].
        On retryable failure: raise RetryableIngestError; Celery will retry.
        On permanent failure: run.status = 'FAILED' with a structured reason.
        """
```

#### 4.2.1 REST adapter (8 vendors)

- Driven by per-vendor `RESTConfig`: base URL, auth strategy (Bearer, HMAC,
  Basic — needs to cover all 8 vendors, see Open Questions on which auth
  per vendor), pagination strategy enum (`offset`, `cursor`, `link_header`),
  request shape, response JSONPath for the items array.
- HTTP client: `httpx` with explicit per-vendor `Timeout(connect=5,
  read=30, total=120)` and retries via `tenacity` (exponential backoff,
  3 attempts on 5xx / connection errors; no retry on 4xx except 429).
- **Rate limiting:** per-vendor token bucket in Redis (we already run
  Redis for Celery). Critically, Vendor F's 60 req/hour cap means we have
  to choose between:
  - one big paginated pull a few times a day (preferred), or
  - smaller deltas if Vendor F supports `updated_since` (TBD; see Open
    Questions). Default plan: 2x daily full pulls for Vendor F, gives us
    ~30 reqs each, comfortably under cap with headroom for retries.
- Per-vendor concurrency: at most 1 in-flight run per vendor (enforced
  via `SELECT FOR UPDATE SKIP LOCKED` on `VendorRun` or a Redis lock).
- Response body streamed to S3 page-by-page as it's fetched, so a
  vendor's pagination state is recoverable.

#### 4.2.2 SFTP adapter (3 vendors)

- Runs on a Celery Beat schedule per vendor (e.g. "every day at 02:30
  UTC, look for today's file").
- Library: `paramiko` (already in many Django codebases; battle-tested).
  Connect with key-based auth where possible, password as fallback,
  credentials in AWS Secrets Manager.
- Behavior on connect:
  - List the configured remote directory.
  - Match files against a per-vendor glob (`vendor_b_inventory_*.csv`).
  - For each new file (not in our dedupe ledger, see 4.5): download to
    a tempfile, checksum (SHA-256), upload to S3, write a `VendorRun`,
    enqueue the pipeline task.
  - Optionally move the file to a `processed/` subdir on the SFTP (per
    vendor — some vendors will not allow writes; config flag).
- "File didn't show up by 04:00 UTC" is itself a signal — Beat runs a
  per-vendor `sftp_check` task at the deadline that asserts a run for
  today exists; if not, it raises a `VendorMissingDataAlert`.

#### 4.2.3 Email adapter (Vendor K — the messy one)

This is the part of the design that has to be *honest*. Vendor K sends an
email "daily-ish", from "usually but not always the same address", with an
attachment that is "usually but not always a CSV" and "usually but not
always named the same thing". We design for that, not against it.

Receiving infrastructure:

- A dedicated email address: `vendor-k-inventory@inventory.ourdomain.com`,
  routed via SES inbound rules to `s3://inventory-sync-inbound-email/`.
- SES rule triggers an SNS -> SQS fanout; a Celery task `consume_inbound_email`
  polls SQS.
- For each message:
  1. Parse MIME; persist the raw `.eml` to S3 alongside extracted parts.
  2. Apply **acceptance rules** (per-vendor config, but for K specifically):
     - From-domain allowlist (any address `@vendor-k.com`, plus a known
       list of personal Gmail addresses Karen-from-vendor-K sometimes
       uses — yes, really).
     - Subject regex (loose: `inventory|stock|levels`).
     - Attachment present with one of: `.csv`, `.xlsx`, `.zip`.
  3. If any acceptance rule fails: file the email under
     `quarantine/` in S3, write a `VendorRun` with status `QUARANTINED`,
     and Slack-ping the ops channel. Do **not** silently drop.
  4. If accepted: extract the attachment, write it to the raw bucket,
     create a `VendorRun` with status `INGESTED`, enqueue pipeline.
- We also support a **manual override**: ops can drag an attachment into
  the Django admin and trigger a `VendorRun` themselves. This is the
  release valve for "Karen sent it as a Google Sheets link this week".

Design notes specifically for Vendor K:

- We do **not** try to make the email path look like the API path to the
  rest of the system — it has its own state machine and its own SLA
  ("file landed in inbox by 09:00 ET; if not, page ops, not on-call eng").
- DKIM/SPF results are recorded but not used to reject — Karen's
  personal Gmail will fail SPF for the vendor domain. Quarantine on
  acceptance-rule failure is sufficient.
- We accept that Vendor K will fail more often. The staleness alert for
  K is tuned to 30h (vs 24h default) to reduce false pages, and routed
  to `#ops-supply`, not `#eng-oncall`. K is 3% of revenue; we are
  explicit that human-in-the-loop is the correct level of automation
  for this vendor.

### 4.3 Parse + validate

Per-vendor parser, selected by `vendor.source_format`:

- `json` (REST): JSONPath to items array; per-field path map.
- `csv`: `csv.DictReader` with per-vendor delimiter / quoting / encoding
  config. Use `csvkit`-style robust parsing — we already have csvkit
  patterns in-house we can pull from.
- `xlsx`: `openpyxl` read-only mode.

Each parser produces an iterator of `RawRecord` dicts. Validation runs
per-record:

- Required fields present (`sku`, `qty`).
- Types coerce (`qty` is non-negative integer; `sku` is non-empty string).
- A per-vendor `record_validator` hook (optional) for vendor-specific
  weirdness (e.g. Vendor C uses "Y/N" for "available" instead of a
  number — we convert to 0 / very-large-int with documented semantics).

Validation failures: collected, capped at 1000 per run, written to
`VendorRun.validation_errors` JSONB. A run with >5% record failures is
marked `PARTIAL` and does **not** auto-apply (see 4.6).

### 4.4 Normalize to internal SKU model

The internal model the catalog expects (already exists):

```
InventoryUpdate {
  sku: str           # our internal SKU
  on_hand: int       # quantity available
  vendor_id: str
  source_observed_at: datetime  # when the vendor said this was true
}
```

Normalization responsibilities:

- Map vendor SKU -> our SKU via `SkuMapping` table
  (`(vendor_id, vendor_sku) -> internal_sku`). Unknown vendor SKUs are
  recorded in `UnknownSku` table and surfaced in a daily ops digest;
  they do **not** fail the run. (PRD calls this out explicitly.)
- Apply per-vendor `qty_transform` (e.g. Vendor D reports "case packs of
  6"; we multiply).
- Drop records where `on_hand` is clearly nonsense (>1M units) and
  count them as validation errors.

### 4.5 Diff vs catalog

- Pre-fetch current catalog state in one bulk read per run:
  `GET /catalog/inventory?vendor_id=X` returning `{sku: on_hand}` (we'll
  need a small addition to catalog's API — confirmed feasible with
  catalog team in pre-design conversation; see Open Questions).
- In memory, compute the diff:
  - **changes** = records where `new.on_hand != current.on_hand`.
  - **unchanged** = skipped.
  - **new** = vendor reports a SKU we have a mapping for but catalog
    has no row → treat as a change (apply).
  - **dropped** = catalog has a row for this vendor that the new feed
    doesn't mention. Handling depends on `vendor.dropped_sku_policy`:
    - `ignore` (default — safest, vendors with partial feeds).
    - `mark_zero` (vendor sends complete daily snapshot; missing means 0).
    - `flag_only` (write audit row, do not change catalog).
- Diff result is persisted (`VendorRunDiff`) before apply, so we can
  inspect what *would* have happened in shadow mode.

### 4.6 Sanity gates before apply (the "messy cases" bullet from PRD)

A run only proceeds to Apply if all of:

- `run.status in (INGESTED, PARTIAL with operator override)`.
- Diff sanity checks:
  - **Empty / zero-inventory guard**: if >=50% of this vendor's SKUs go
    to 0 in one run, *block the apply* and alert. This catches the
    "vendor briefly returns empty inventory" failure mode the PRD
    explicitly mentions. Operator can override via admin if it's
    legitimately Black Friday and they really did sell out.
  - **Change-rate guard**: if the diff would change >25% of the vendor's
    SKUs (configurable), block and alert. Catches "field mapping
    broke" and "vendor changed units silently".
- Vendor `apply_enabled = True` (the shadow-mode lever).

A blocked run is not retried automatically — it needs human ack.

### 4.7 Apply

- Catalog already exposes `POST /catalog/inventory/bulk_update` accepting
  up to 1000 items per call. We chunk and call with idempotency keys
  `{run_id}:{chunk_index}`. The catalog team confirms they store keys
  and dedupe.
- Concurrency: at most one Apply per vendor at a time (enforced upstream;
  also enforced by catalog's per-(vendor, sku) row locking).
- On 5xx or timeout from catalog: retry with backoff (3 attempts).
  Persistent failure → run is `APPLY_FAILED`; alert; the next scheduled
  run will pick up the same state from vendor and try again. We do
  **not** partial-apply — either a chunk's bulk_update succeeds, or
  we retry the chunk. (Catalog's idempotency keys make this safe.)
- Every applied record produces one row in `InventoryAuditLog`.

### 4.8 Staleness detection

A Celery Beat job runs every 15 minutes:

```sql
SELECT vendor_id, MAX(finished_at) AS last_ok
FROM vendor_run
WHERE status = 'APPLIED'
GROUP BY vendor_id
HAVING MAX(finished_at) < NOW() - vendor.staleness_threshold;
```

For each row: emit a `vendor.stale` event (Datadog metric +
`inventory_sync.stale_vendor_total` counter; PagerDuty for vendors with
`alert_severity = sev2`, Slack for everyone else).

Stale check is *separate* from "the last run failed" alerts — both can
fire, and they mean different things.

---

## 5. Data model (Postgres)

Owned by inventory-sync service. Catalog DB is not touched directly.

```sql
-- Per-vendor configuration. Editable via admin; versioned via Django audit.
CREATE TABLE vendor (
    id                      TEXT PRIMARY KEY,        -- 'vendor_a' .. 'vendor_l'
    name                    TEXT NOT NULL,
    source_type             TEXT NOT NULL,           -- 'rest' | 'sftp' | 'email'
    source_format           TEXT NOT NULL,           -- 'json' | 'csv' | 'xlsx'
    config                  JSONB NOT NULL,          -- adapter-specific (URL, paths, auth ref)
    field_map               JSONB NOT NULL,          -- {our_field: vendor_field_or_path}
    schedule_cron           TEXT,                    -- NULL for email (event-driven)
    rate_limit_per_hour     INT,                     -- e.g. 60 for Vendor F
    staleness_threshold     INTERVAL NOT NULL DEFAULT '24 hours',
    dropped_sku_policy      TEXT NOT NULL DEFAULT 'ignore',
    apply_enabled           BOOLEAN NOT NULL DEFAULT FALSE,  -- shadow mode toggle
    empty_guard_pct         INT NOT NULL DEFAULT 50,
    change_guard_pct        INT NOT NULL DEFAULT 25,
    alert_severity          TEXT NOT NULL DEFAULT 'sev3',
    enabled                 BOOLEAN NOT NULL DEFAULT TRUE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- One row per attempt to pull a vendor.
CREATE TABLE vendor_run (
    id                      UUID PRIMARY KEY,
    vendor_id               TEXT NOT NULL REFERENCES vendor(id),
    trigger                 TEXT NOT NULL,        -- 'scheduled' | 'email' | 'manual' | 'replay'
    status                  TEXT NOT NULL,
    -- statuses: PENDING, INGESTING, INGESTED, PARSED, DIFFED, BLOCKED,
    --          APPLIED, APPLY_FAILED, FAILED, QUARANTINED, PARTIAL
    started_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at             TIMESTAMPTZ,
    raw_s3_keys             TEXT[],
    record_count            INT,
    change_count            INT,
    validation_errors       JSONB,
    block_reason            TEXT,
    error                   JSONB,
    -- For email-triggered runs:
    inbound_email_id        TEXT,
    UNIQUE (vendor_id, trigger, started_at)
);
CREATE INDEX ON vendor_run (vendor_id, status, finished_at DESC);

-- Pre-apply diff snapshot. Always written, even in shadow mode.
CREATE TABLE vendor_run_diff (
    run_id                  UUID PRIMARY KEY REFERENCES vendor_run(id),
    changes                 JSONB NOT NULL,       -- [{sku, old, new}]
    new_skus                JSONB,
    dropped_skus            JSONB,
    summary                 JSONB                 -- counts + guard outcomes
);

-- Append-only audit of every change actually pushed to catalog.
CREATE TABLE inventory_audit (
    id                      BIGSERIAL PRIMARY KEY,
    run_id                  UUID NOT NULL REFERENCES vendor_run(id),
    vendor_id               TEXT NOT NULL,
    sku                     TEXT NOT NULL,
    old_on_hand             INT,
    new_on_hand             INT NOT NULL,
    applied_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    source_observed_at      TIMESTAMPTZ
);
CREATE INDEX ON inventory_audit (sku, applied_at DESC);
CREATE INDEX ON inventory_audit (vendor_id, applied_at DESC);

-- Vendor SKU <-> internal SKU.
CREATE TABLE sku_mapping (
    vendor_id               TEXT NOT NULL,
    vendor_sku              TEXT NOT NULL,
    internal_sku            TEXT NOT NULL,
    qty_multiplier          INT NOT NULL DEFAULT 1,
    PRIMARY KEY (vendor_id, vendor_sku)
);

-- Unknown vendor SKUs seen during a run; surfaced for ops triage.
CREATE TABLE unknown_sku (
    vendor_id               TEXT NOT NULL,
    vendor_sku              TEXT NOT NULL,
    first_seen_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    seen_count              INT NOT NULL DEFAULT 1,
    PRIMARY KEY (vendor_id, vendor_sku)
);

-- Dedupe ledger for SFTP / email payloads (so we don't reprocess
-- the same file if a vendor uploads it twice).
CREATE TABLE payload_dedupe (
    vendor_id               TEXT NOT NULL,
    sha256                  TEXT NOT NULL,
    first_seen_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (vendor_id, sha256)
);
```

Retention:

- `vendor_run`, `vendor_run_diff`: 180 days hot in Postgres, archived to S3.
- `inventory_audit`: 2 years hot (finance asks for it); partitioned monthly.
- `unknown_sku`: indefinite, small.
- S3 raw payloads: 1 year, lifecycle-rule to Glacier after 90 days.

---

## 6. API contracts

### 6.1 Calls *we make* to catalog (existing API, small additions)

- `GET /catalog/inventory?vendor_id={vid}&fields=sku,on_hand`
  - **Existing? Confirm.** If not, this is a small ask of the catalog
    team — single endpoint, paginated, returns the current state for one
    vendor's SKUs. (Open Question.)
- `POST /catalog/inventory/bulk_update`
  - Body: `{ "idempotency_key": "...", "items": [ {sku, on_hand, source_observed_at} ] }`
  - Already exists per PRD ("has a REST update API"); we'll confirm
    idempotency-key support exists, or coordinate adding it.
  - Response: 200 with per-item ok/failure detail, or 409 if dup key.

### 6.2 Internal HTTP we expose (Django admin + ops)

- `POST /sync/vendors/{vendor_id}/runs` — trigger a manual run (ops).
- `POST /sync/vendors/{vendor_id}/runs/{run_id}/override` — operator
  override of a `BLOCKED` run.
- `POST /sync/vendors/{vendor_id}/runs/{run_id}/replay` — re-run from
  the S3 raw payloads (handy when a parser bug ships).
- `GET /sync/vendors/{vendor_id}/runs?status=...` — list runs.
- `GET /sync/health` — liveness; reports last-success-per-vendor.

All admin endpoints behind existing Django auth + ops role check.

### 6.3 No outbound event bus (yet)

PRD asks "how do we surface discontinued SKU events to storefront?" —
out of scope. The audit log + a future `inventory_changed` event topic
(SNS) is the natural extension; mentioned in Open Questions.

---

## 7. Alternatives considered

| Alternative | Why rejected |
|---|---|
| One adapter per vendor, no shared pipeline | 12 nearly-identical code paths, 12 places to fix any bug. Doesn't scale to "vendor 13". |
| Use AWS Step Functions or Airflow for orchestration | Real cost: another deploy target, another on-call surface, IAM, secrets. We have ~12 schedules and one fan-out. Celery is sufficient and the org already runs it. Revisit if we get past ~50 vendors. |
| Pull straight into the catalog DB | Couples two services at the schema layer. Catalog team is rightly protective of their schema. REST API exists; use it. |
| Stream directly into catalog without S3 staging | Loses the ability to replay parser bugs. Costs ~$50/mo in S3. Trivially worth it. |
| Have ops paste Vendor K's CSV into a UI every day | This is essentially the status quo and the project exists to remove it. We do, however, keep the **manual override** in the admin for when SES inbound genuinely fails. |
| Ask Vendor K to switch to SFTP | Tried; per PRD, non-negotiable, take it or leave it. They're 3% of revenue. Build for what is, not what we wish. |
| Webhook ingress (vendor pushes to us) | Only 1 of 12 vendors offers webhooks; not worth a fourth ingestion shape for one optional source. Listed as a future option. |

---

## 8. Non-functional requirements

### 8.1 Capacity

- ~140K SKUs total across 12 vendors → average ~12K SKUs/vendor.
- Vendor data volume per run: <= ~10 MB CSV, <= ~50 MB JSON pages (worst-case
  Vendor A). Wholly fits in memory; pipelines stream where it doesn't cost
  much, but no special big-data tooling required.
- Diff per run: O(N) on 12K rows = trivial.
- Catalog bulk_update: 12K SKUs / 1000 per call = ~12 calls per vendor per
  run. At 12 vendors * (1–8 runs/day) = at most ~1200 catalog calls/day.
  Well within catalog's existing capacity.
- Peak concurrency: <= 12 ingestion tasks in-flight (one per vendor) +
  pipeline tasks. Two Celery worker boxes (4 vCPU, 8 GB) sized
  generously cover this with headroom.

### 8.2 Performance

- End-to-end SLO per source type stated in section 2.
- Per-vendor latency budget:
  - Ingest: <= 5 min (REST poll), <= 5 min (SFTP fetch), <= 5 min
    (email parse).
  - Parse + validate: <= 2 min.
  - Diff: <= 1 min.
  - Apply: <= 5 min (12 chunks * ~5 s catalog roundtrip + retries).

### 8.3 Reliability and availability

- Target service availability for inventory-sync: 99.5% monthly. This is
  a batch / near-batch system; brief unavailability does not cause
  customer-facing impact (the catalog and storefront are independent).
- Idempotent everywhere: re-running an `INGESTED` run from S3 must
  produce the same diff and the same apply.
- Failure of one vendor never blocks another (per-vendor queues + locks).
- Retries: 3 attempts with exponential backoff at the adapter layer; the
  scheduler picks up the next run on schedule regardless.

### 8.4 Consistency

- Catalog is eventually consistent vs vendor. We never claim live data.
- Within a single run, apply is all-or-each-chunk: a chunk either fully
  succeeds (with idempotency) or we retry that chunk. We do not promise
  full transactionality across a 12K-SKU run; that is acceptable for
  this domain (a partially-applied run leaves catalog more accurate than
  before, not less, and the next run completes the rest).

### 8.5 Security

- Vendor credentials in AWS Secrets Manager, fetched at task start, never
  written to logs. `config` JSONB stores *references* to secrets, not
  secret values.
- SFTP: prefer key-based auth; keys provisioned per vendor.
- Email inbound:
  - Dedicated subdomain (`inventory.ourdomain.com`) so the rest of the
    org's email isn't in scope.
  - From-domain allowlist gates which inbound emails create runs.
  - Attachments scanned for size limits (reject >50 MB) and MIME-type
    sniffed independently of file extension.
  - **Antivirus scanning** of attachments via ClamAV (lambda hook on
    S3 inbound bucket) before any worker reads them.
- Outbound to catalog: mTLS or signed JWT — whatever catalog already
  requires.
- Network: all components stay in VPC; only SES inbound is internet-
  facing, and SES handles that.
- PII: vendor inventory data is not PII. Email metadata (sender) is
  retained in audit; not a privacy concern.
- Audit log is append-only at the application layer; DB user has
  `INSERT, SELECT` only on `inventory_audit`.

### 8.6 Observability

Metrics (Datadog, prefixed `inventory_sync.`):

- `runs_total{vendor, status}` — counter.
- `run_duration_seconds{vendor, stage}` — histogram.
- `records_total{vendor}` — gauge per run.
- `changes_total{vendor}` — gauge per run.
- `validation_errors_total{vendor}` — counter.
- `catalog_apply_latency_seconds` — histogram.
- `vendor_freshness_seconds{vendor}` — gauge of `now() - last_apply_ok`,
  scraped every 15 min. Powers the staleness alert.
- `inbound_emails_total{result}` — counter (`accepted | quarantined | rejected`).

Logs:

- Structured JSON, one `run_id` carried through every log line in a run.
- Vendor adapter logs at INFO; record-level validation failures at DEBUG
  (sampled at INFO).
- Raw vendor payloads NEVER logged; they're in S3.

Tracing:

- OpenTelemetry spans wrapping each stage of the pipeline; vendor and
  run_id as span attributes.

Dashboards (one Datadog dashboard, sections):

- Per-vendor freshness heatmap.
- Per-vendor run-success rate (7d).
- Validation-error trend.
- Apply latency p50/p95/p99.
- Blocked runs awaiting operator ack.

Alerts (severity):

- `sev2`: `vendor_freshness_seconds{vendor=X} > 24h` for any vendor with
  `alert_severity=sev2`. Pages on-call.
- `sev3`: same condition for sev3 vendors (Slack only).
- `sev2`: `runs_total{status=APPLY_FAILED}` > 0 in 15 min.
- `sev3`: `inbound_emails_total{result=quarantined}` > 0 in 1h.
- `sev2`: empty-inventory guard tripped for a vendor (block + alert).
- `sev3`: change-rate guard tripped.

### 8.7 Scalability

- Adding a vendor is configuration only as long as the source falls into
  existing adapter shapes. New auth schemes or new source formats are
  small adapter changes.
- 140K SKUs / 12 vendors / a few runs a day is far from any architectural
  ceiling. The bound is human (the count of vendor onboarding tasks),
  not technical.
- If we ever hit 50+ vendors or 1M SKUs: shard Celery workers by vendor
  ID, partition `inventory_audit` by month (already planned), revisit
  the Beat -> dynamic-schedule pattern.

### 8.8 Cost

- AWS: low. EC2/Fargate for two worker boxes (~$120/mo), Postgres rows
  are tiny, S3 raw bucket (~few GB/month, <$5), SES inbound is cents.
  Estimate <$300/mo all-in for infra.
- Engineering: 2 engineers * 6 weeks ≈ 12 engineer-weeks. PRD bound.

---

## 9. Failure modes and how the design handles them

| Failure | Detection | Mitigation |
|---|---|---|
| Vendor REST API returns 5xx | HTTP layer | 3 retries with backoff; run -> FAILED; next scheduled run picks up; staleness alert fires if it stays broken. |
| Vendor F rate limit hit | 429 from vendor | Token-bucket throttle prevents in normal operation; on 429, back off respecting `Retry-After` header. |
| Vendor returns empty inventory ("blank world") | `empty_guard_pct` check pre-apply | Apply blocked, sev2 alert, operator decides. |
| Vendor changes field names / units silently | `change_guard_pct` or validation errors | Apply blocked, alert; new mapping pushed via admin; replay run from S3. |
| SFTP file doesn't appear | Deadline check task | Run with `status=FAILED, reason=no_file`; staleness alert when 24h crossed. |
| SFTP file is a partial / corrupt CSV | Parser errors > 5% | Run -> `PARTIAL`, no apply, alert. |
| Vendor K sends a Google Sheets *link* instead of an attachment | Acceptance rule (no attachment) | Quarantined, ops Slack ping; ops uses admin manual-upload. |
| Vendor K sends from a different address | From-domain allowlist miss | Quarantined; ops can add address to allowlist (config). |
| Vendor K sends *two* emails on the same day | dedupe ledger (SHA-256) | Identical attachment: second one is a no-op. Different attachment: both processed; later run wins (idempotent on catalog side). |
| Unknown SKU in feed | Normalization step | Recorded in `unknown_sku`, surfaced in daily digest; does **not** fail run. |
| Catalog API down | HTTP timeout / 5xx | Retries; persistent failure -> `APPLY_FAILED`, sev2 alert; next run retries the same change set safely because catalog is idempotent. |
| Catalog accepts then loses the update | Reconciliation job (sec 11) | Daily diff between `inventory_audit` and catalog state, alert on drift. |
| Worker crashes mid-run | Celery + DB state | Run row left in non-terminal state; sweeper task moves >2h-stale non-terminal runs to `FAILED`; next scheduled run picks up. |
| S3 write fails | adapter error | Retry; if S3 itself is out, the whole AWS region is having a bad day and inventory sync is correctly the lowest of our worries. |
| Operator triggers two manual runs at once | Per-vendor lock | Second one blocks or fails with `vendor_locked`. |
| Mapping table missing for a new SKU | normalization | Vendor sku recorded in `unknown_sku`; ops adds mapping; replay run. |
| Shadow mode "the new system disagrees with the Google Sheet" | Comparison report job | Daily job that diffs `vendor_run_diff` against the ops sheet (read-only) and posts deltas to `#ops-supply` for human review. (See Open Q.) |

---

## 10. Rollout and migration

### Phase 0 — pre-work (week 1)

- Catalog API confirmations: `bulk_update` idempotency-key support;
  per-vendor `GET` query. Coordinate any catalog-side additions.
- Provision: Postgres DB, Redis, S3 buckets, SES inbound subdomain,
  SecretsManager secrets, IAM roles, Datadog integration.
- Vendor credential collection (12 vendors × N artifacts). Realistically
  the long-pole task; start it on day 1.

### Phase 1 — skeleton + first vendor (weeks 2–3)

- Build core: `vendor`, `vendor_run`, audit tables; pipeline scaffold;
  catalog client; admin UI minimum.
- Onboard one REST vendor end-to-end (pick the easiest, e.g. modern
  JSON API with clean docs). Ship to staging.
- Build the SFTP adapter; onboard one SFTP vendor.
- Build the email adapter; onboard Vendor K (yes, early — its quirks
  will shape the abstraction, and we should not discover them in week 6).

### Phase 2 — fan out (weeks 3–5)

- Onboard remaining 9 vendors, one per ~half-day where possible.
  Most of this is config + field-map work, not code.
- All vendors run in **shadow mode** (`apply_enabled=False`).
- Build reconciliation + staleness alerting.

### Phase 3 — shadow mode (the 2 weeks from PRD)

- All 12 vendors run on schedule, computing diffs, writing
  `vendor_run_diff`, alerting on guard trips — but **not** applying.
- Daily comparison report: "today, new system would have changed X SKUs;
  Google Sheet recorded Y changes; here are the deltas". Ops reviews.
- Tune `empty_guard_pct`, `change_guard_pct`, `staleness_threshold`
  per vendor based on observed behavior.

### Phase 4 — cutover (week 6)

- Flip `apply_enabled=True` per vendor, **in waves**:
  - Day 1: 2 lowest-risk vendors (clean REST, low SKU count).
  - Day 3: next 4.
  - Day 5: next 4.
  - Day 7: Vendor F (rate-limited) and Vendor K (email).
- Monitor oversell-incident metric vs baseline.
- Decommission Google Sheet workflow at end of week (PM owns; we keep
  the audit log + reconciliation as the new source of truth).

### Rollback

- Per-vendor: flip `apply_enabled=False`. No data migration needed; the
  Google Sheet workflow can resume on demand for that vendor.
- Whole-system: disable Celery Beat scheduled tasks. The catalog is
  untouched by inventory-sync going dark; only freshness suffers.

---

## 11. Operations

- **Runbook** (lives in our ops wiki) for each of:
  - Vendor X is stale → check last run, check vendor's status page,
    replay from S3 if it was a parser issue.
  - Quarantined email → admin link, decide accept/reject.
  - Blocked run (guard tripped) → review `vendor_run_diff`, override
    or wait for next run.
  - Catalog apply failing → check catalog service status, retry, page
    catalog team if persistent.
- **Daily reconciliation job** (Celery Beat, 03:00 UTC):
  - Pull catalog state per vendor.
  - Compare against the running sum of `inventory_audit` per (vendor, sku).
  - Any drift > 0.1% of SKUs → sev2 alert. (Bookkeeping bug somewhere.)
- **Weekly digest** to `#ops-supply`:
  - Per-vendor run counts, failure rate, top unknown SKUs.
  - Guard trips and operator overrides for the week.
- **Manual replay** is a first-class operation; ops should be using it
  weekly without engineering involvement.

---

## 12. Testing strategy

- **Unit tests** per adapter (REST/SFTP/email), per parser, per
  normalizer, per guard. Mock vendor sources via fixtures.
- **Golden-file tests**: for each vendor, freeze a representative raw
  payload in `tests/fixtures/vendors/<vendor>/` and assert the full
  pipeline output (records, diff, apply calls). Updating these is a
  conscious code-review step.
- **Integration tests** against:
  - A localstack S3 + SES for the email path.
  - A docker-composed Postgres + Redis.
  - A mock catalog server (`respx`/`pytest-httpx`).
- **Vendor K specifically**:
  - Test cases for: correct email, wrong-from email, no-attachment
    email, two-emails-same-day, attachment-is-xlsx, attachment-is-zip-
    of-csv, attachment-named-`final_v2_REALLY_FINAL.csv`,
    1MB-of-garbage-attachment.
- **Load test**: simulate all 12 vendors firing simultaneously; verify
  per-vendor isolation.
- **Chaos**: take catalog API offline mid-apply; verify retry + audit.

---

## 13. Open questions

These need answers from PM, catalog team, vendors, or ops before or
during implementation. Listed in priority order.

1. **Catalog API surface.** Does `bulk_update` already accept
   idempotency keys? Does a `GET /inventory?vendor_id=X` exist, or do
   we need it added? (Owner: catalog tech lead.) Blocks 4.5 + 4.7.
2. **Shadow-mode arbitration (PRD open item).** When the new system and
   the Google Sheet disagree during shadow mode, who arbitrates? Our
   recommendation: ops owns it, eng provides the diff report; PM signs
   off cutover per vendor.
3. **Discontinued-SKU events (PRD open item).** Out of scope here, but
   we propose an SNS topic `inventory.sku_status_changed` as a follow-up
   project. Storefront team should be a stakeholder if/when we do it.
4. **Per-vendor auth schemes** for the 8 REST vendors — we'll need a
   per-vendor inventory of auth (Bearer / HMAC / OAuth client-credentials
   / Basic) before deciding the exact REST adapter abstraction.
5. **Vendor F rate-limit strategy:** does Vendor F support
   `updated_since` / delta queries, or must we always do full pulls?
   Affects whether 60 req/hour is comfortable.
6. **Dropped-SKU policy per vendor** — for each of the 12 vendors, do
   they send a full snapshot or a delta? PM + ops to confirm. Defaults
   to `ignore`, but `mark_zero` is correct for full-snapshot vendors
   and we don't want to learn that the hard way.
7. **Vendor K acceptance details:** confirm the inventory of personal
   email addresses Karen uses; confirm whether `.xlsx` is allowed (or
   if she always sends `.csv`); confirm the time window in which her
   email is "expected" so we can tune the missing-data alert.
8. **Ops digest delivery:** Slack channel and digest cadence — proposed
   `#ops-supply`, daily 09:00 ET.
9. **PII / compliance review:** confirm with security that retaining
   inbound emails (with sender PII) for a year is acceptable, and what
   the deletion process should look like.
10. **Storage retention:** confirm 180-day hot / 2-year audit / 1-year
    raw S3 numbers with finance and compliance.
11. **What happens on the day Vendor K sends *zero* inventory across the
    board?** Empty-inventory guard catches it — but is the right answer
    "block and ping ops" or "interpret as legitimately sold-out"? Per
    PRD's "vendor briefly returns empty inventory" mention, we default
    to block + ping; confirm with PM.

---

## 14. Appendix: vendor matrix template

To be filled in during phase 0 (one row per vendor; this is the artifact
that turns 12 vendors into 12 onboarding tickets):

| Vendor | Mechanism | Format | Auth | Cadence | Rate limit | Full snapshot? | Notes |
|---|---|---|---|---|---|---|---|
| A | REST | JSON | Bearer | hourly | none stated | delta | clean docs |
| B | SFTP | CSV | key | nightly 02:00 UTC | n/a | full | tab-delimited, watch out |
| C | REST | JSON | HMAC | every 4h | 1000/hr | full | uses Y/N for availability |
| ... | ... | ... | ... | ... | ... | ... | ... |
| F | REST | JSON | OAuth2 | 2x daily | 60/hr | TBD | the rate-limited one |
| ... | ... | ... | ... | ... | ... | ... | ... |
| K | Email | CSV(usually) | from-domain allowlist | "daily-ish" | n/a | full | Karen; small supplier; 3% of revenue |
| L | REST | JSON | Basic | hourly | 600/hr | delta | |
