# TDD: Vendor Inventory Sync

**Author:** Tech Lead (drafted on behalf of the user)
**Status:** Draft
**Date:** 2026-06-09
**Related PRD:** `/Users/nhoze/.claude/skills/tech-lead/evals/prd-vendor-inventory-sync.md` (Priya Subramanian, 2026-05-30)

> Note on project conventions: no `CLAUDE.md` / `CONTRIBUTING.md` / `ARCHITECTURE.md` was provided for the target system. Constraints in the PRD (Python/Django/Celery shop, AWS us-east-2, existing Postgres-backed catalog service with REST update API) are treated as authoritative; section 0 is omitted because there is no repo-level convention document to cite.

---

## 1. Problem & context

We sell ~140K SKUs across 12 vendors whose inventory is currently re-entered into a Google Sheet by hand and pushed to the catalog manually. The sheet is at least 24h stale and frequently wrong, costing ~$80K/month in "out of stock after order placed" incidents (per Q1 CX report). The fix is an automated sync that pulls each vendor's inventory at a vendor-appropriate cadence, diffs it against the catalog, and applies changes.

The catch is the ingestion fan-in: **8 vendors expose REST APIs we can poll, 3 drop a nightly CSV onto SFTP, and 1 vendor (K, ~3% of revenue) emails us a daily CSV attachment.** Vendor K is non-negotiable on delivery mechanism. So this is *not* one workload — it's three ingestion shapes feeding one normalization-and-apply pipeline. The design has to be honest about that, not paper over it with "we'll write an adapter".

The catalog service is an existing internal REST service in the same VPC; we don't own its DB. Storefront reads from the catalog. We are the writers.

Driver: revenue loss is ongoing and quantified. PM has committed to a 2-week shadow-mode validation before cutover, so the engineering deadline is "shadow mode running for all 12 vendors" within the 6-week, 2-engineer budget.

## 2. Scope

**In scope**
- Per-vendor sync of inventory for all 12 vendors, including the email-attachment vendor.
- Three ingestion adapters: REST poll, SFTP poll, inbound-email handler.
- Normalization from vendor-specific schemas into a single internal `VendorInventorySnapshot` shape.
- Diff against current catalog state and a write path that calls the catalog service's existing REST update API.
- Audit log of every applied inventory change (SKU, old qty, new qty, source vendor, source timestamp, ingestion run id).
- Freshness monitoring per vendor with alerting when no successful pull in > 24h (or vendor-specific threshold).
- Shadow mode — compute proposed changes, log them, do *not* call the catalog write API.
- Per-vendor configuration (source type, credentials, cadence, field mapping, rate-limit hints).

**Out of scope**
- Two-way sync (we never push our sales data to vendors).
- Real-time / sub-hour freshness. Best vendor data is hourly; we target "fresh enough", not "live".
- Replacing the Google Sheet workflow on day 1 — shadow mode for 2 weeks first.
- Discontinued-SKU pipeline to the storefront team (see PRD open question; tracked in §10).
- Vendor onboarding self-service UI. Adding a new vendor in the first release is a config + code change.

## 3. High-level approach

Three thin **ingestion adapters** (REST, SFTP, Email) each land raw vendor payloads as immutable blobs in S3 and enqueue a normalization job. A single **normalization-and-apply pipeline** in Celery reads the blob, parses + validates against a per-vendor schema, normalizes to `VendorInventorySnapshot`, diffs against the most recent applied snapshot for that vendor, and emits per-SKU updates to the catalog service. Cursor / watermark state, per-vendor config, and audit log live in a small Postgres DB owned by this service (separate from the catalog DB).

The orchestration is **Celery Beat** for scheduled REST and SFTP polling; the email vendor is event-triggered by SES → S3 → Lambda → SQS → Celery. All three converge on the same downstream pipeline so the messy parts (validation, diff, write, audit, alerting) are written and tested once.

Diagram (describe in prose; no fabricated link):

```
                                   +--------------------+
   Celery Beat (cron) --(REST)---->| REST adapter       |--+
                                   +--------------------+  |
                                                           |
   Celery Beat (cron) --(SFTP)---->| SFTP adapter       |--+--> S3 raw landing
                                   +--------------------+  |    (s3://vendor-inv-raw/<vendor>/<yyyy>/<mm>/<dd>/<run_id>.<ext>)
                                                           |
   SES inbound rule --> S3 + SNS ->| Email adapter      |--+
                                   +--------------------+         |
                                                                  v
                                                    SQS: normalize-queue
                                                                  |
                                                                  v
                                                   +---------------------------+
                                                   | Celery worker pool        |
                                                   |  - parse + validate       |
                                                   |  - normalize              |
                                                   |  - load last snapshot     |
                                                   |  - diff                   |
                                                   |  - apply via catalog API  |
                                                   |  - write audit rows       |
                                                   +---------------------------+
                                                                  |
                                                                  v
                                            Postgres (this service):
                                              vendor_config,
                                              vendor_run, vendor_cursor,
                                              vendor_snapshot, inventory_audit
                                                                  |
                                                                  v
                                                   Catalog service REST API
                                                   (existing, Postgres-backed)
```

The key insight: **the source mechanism varies; everything downstream of "raw payload in S3 + run row in Postgres" is uniform.** That uniformity is what keeps the design honest about the 1-CSV-by-email vendor — it doesn't fork the pipeline, only the adapter.

## 4. Detailed design

### 4.1 Per-vendor configuration

A `vendor_config` table (or YAML loaded at startup; see §10 OQ-1). One row per vendor with:

- `vendor_id` (string slug, e.g. `vendor_a`, `vendor_k`)
- `source_type` enum: `rest` | `sftp` | `email`
- `cadence_cron` (for REST/SFTP) — e.g. `*/15 * * * *`
- `rate_limit_per_hour` (advisory; respected by the REST adapter's token bucket)
- `source_config` (JSONB) — endpoint URL / SFTP host / expected sender address; per-type schema
- `secret_ref` — pointer to AWS Secrets Manager ARN
- `field_mapping` (JSONB) — vendor field name → internal field, plus type coercion rules
- `staleness_threshold_minutes` — default 1440 (24h), overridable per vendor
- `enabled` boolean and `shadow_mode` boolean (per-vendor cutover flag)

### 4.2 Ingestion adapters

The three adapters share one job: produce a `vendor_run` row (`PENDING` status, run_id, vendor_id, ingestion_started_at, raw_s3_key) and enqueue an SQS message `{run_id}` onto `normalize-queue`. They do **not** parse, normalize, or write to the catalog.

#### REST adapter (8 vendors)

- Celery Beat schedule kicks `pull_vendor_rest(vendor_id)` per configured cadence.
- Each task acquires a per-vendor advisory lock (Postgres `pg_advisory_lock(hashtext('vendor_rest:' || vendor_id))`) — no two REST runs for the same vendor overlap.
- Token-bucket rate limiter per vendor in Redis (key `rl:rest:<vendor_id>`, refill = `rate_limit_per_hour/3600` per second). **Vendor F is capped at 60 req/hr total** — at a 15-min cadence that's 4 runs/hr → 15 pages/run max. The bucket is the enforcement, not the cadence.
- Pagination cursor (vendor-specific: page token, "since" timestamp, or ETag) is persisted in `vendor_cursor` *after the full page set is written to S3*. Critically: **cursor is advanced after raw landing succeeds, not after normalization succeeds.** Re-normalization of the same raw blob is idempotent (see §4.4); re-pulling the same window from a vendor is not always safe (some vendors charge per call, some have sliding-window quotas).
- Retry policy: 3 attempts with exponential backoff + jitter (1s, 4s, 16s ± 30% jitter), then `vendor_run.status = FAILED` and the freshness alert fires if this puts us over threshold.
- HTTP timeout: 30s connect, 60s read. No unbounded waits.
- Auth failure (401/403) does **not** retry — it pages immediately. Credential rotation is the most common cause and silent retries delay detection.

#### SFTP adapter (3 vendors)

- Celery Beat schedule kicks `pull_vendor_sftp(vendor_id)` daily (nightly drops).
- Uses `paramiko` over an SSH key stored in Secrets Manager.
- Lists `INBOX/` (or vendor-specific path), filters to new files (by name pattern + mtime newer than `vendor_cursor.last_seen_mtime`), downloads each to S3, then advances cursor to the max mtime seen.
- A file is considered "new" *only after a successful upload to S3*; partial transfers leave the cursor unchanged.
- One `vendor_run` per file. Multiple new files on the same tick = multiple runs.
- "No new file" is a successful run (status `EMPTY`), recorded with its own timestamp — distinct from `FAILED`. This is the distinction the PRD explicitly asks for ("a vendor's API is down" vs "vendor briefly returns empty").

#### Email adapter (Vendor K only)

This is the part the PRD explicitly flagged. The design choice: **do not try to fit email into the polling shape**. Push, not pull.

- AWS SES inbound receiving rule for a dedicated address (e.g. `vendor-k-inventory@inbound.<our-domain>`).
- SES rule action: write the full raw MIME message to S3 (`s3://vendor-inv-raw/vendor_k/raw_email/<message_id>.eml`) and publish to SNS.
- SNS → SQS → Lambda (or Celery task) that:
  1. Verifies sender address matches `source_config.expected_sender` (no DKIM/SPF in SES inbound by default — must rely on SES receipt and the from-address verification, see §7 risk R-EMAIL-2).
  2. Walks MIME parts, finds the first attachment whose name matches a configurable regex (default `inventory.*\.csv$`).
  3. Writes the attachment to `s3://vendor-inv-raw/vendor_k/<yyyy>/<mm>/<dd>/<message_id>.csv`.
  4. Creates a `vendor_run` row and enqueues `normalize-queue`.
- **Idempotency on email:** SES can re-deliver. The Message-ID header is the dedupe key (`UNIQUE` constraint on `vendor_run.source_message_id`). A repeated Message-ID is logged and dropped.
- **What if Vendor K sends two emails in one day** (e.g. correction). Both get processed; the second snapshot supersedes the first. The audit log shows both.
- **What if Vendor K's email doesn't arrive by the daily deadline.** Staleness alert fires at `staleness_threshold_minutes` (configurable; suggest 30h for K to absorb their irregular delivery window).
- We do **not** implement: SMTP server, IMAP polling of a regular inbox, attachment OCR. Out of scope.

### 4.3 Normalize-and-apply pipeline

Single Celery task `normalize_run(run_id)` consuming `normalize-queue`:

1. **Load run row.** If `status != PENDING`, log and return (idempotent on redelivery from SQS).
2. **Read raw payload from S3.** Streamed where possible (CSVs >100MB are not expected per vendor but plan for 500MB to be safe).
3. **Validate against per-vendor schema.** Pydantic models defined per vendor. On schema failure: mark `vendor_run.status = VALIDATION_FAILED`, write the validation errors to `vendor_run.error_detail`, alert. Do not partial-apply.
4. **Normalize to internal `VendorInventorySnapshot`.** A list of `(sku, available_qty, vendor_sku_id, vendor_last_updated_at)`. Field mapping comes from `vendor_config.field_mapping`.
5. **SKU resolution.** For each `vendor_sku_id`, look up our internal SKU. Unknown SKU → goes to a `unknown_sku_log` table (not an error; reported in run summary). The PRD calls this out as a messy case.
6. **Load previous applied snapshot** for this vendor from `vendor_snapshot` (`WHERE vendor_id = $1 AND status = APPLIED ORDER BY applied_at DESC LIMIT 1`).
7. **Compute diff.** Per-SKU: added / removed / changed. "Removed" from a vendor snapshot does **not** automatically mean qty=0 — see §4.5 for the "vendor briefly returns empty" guard.
8. **Apply.** For each changed SKU, call catalog service `PUT /catalog/inventory/{sku}` with `{vendor_id, available_qty, source_timestamp}`. Concurrency: 8 in-flight (configurable), per-request timeout 5s, retry 3× with backoff on 5xx.
   - In **shadow mode**, skip the API call and write the proposed change to `inventory_audit` with `applied = false`.
9. **Write audit rows.** One row per SKU change, with `applied = true|false`, `source = vendor_id`, `run_id`, `source_timestamp`, `old_qty`, `new_qty`.
10. **Mark run `APPLIED`** and persist the new `vendor_snapshot` row.

### 4.4 Cursor / watermark and idempotency

- **REST cursor:** vendor-specific; persisted *after raw S3 write succeeds*. Re-running normalization for the same raw blob is idempotent because the diff is computed against the last *applied* snapshot, not the previous raw blob. So re-delivery of a normalize message produces the same diff and the catalog write is itself an idempotent upsert keyed by `(sku)`.
- **SFTP cursor:** max mtime of files successfully uploaded to S3.
- **Email cursor:** N/A — driven by message Message-ID dedup.
- **First run:** for REST vendors that support "since" cursors, start with `now() - vendor_config.initial_lookback` (default 24h). For vendors whose API only returns "current state", first run is just a full pull.
- **Cursor corruption recovery:** cursor stored in Postgres with a daily snapshot to S3. Restoring a cursor is a manual ops action; documented in runbook.

### 4.5 The "vendor briefly returns empty" guard

A real failure mode the PRD calls out and which has caused real production incidents in adjacent systems. Defense in depth:

- If a fetched snapshot has **< 50% of the SKU count of the previous APPLIED snapshot for the same vendor**, mark the run `QUARANTINED` instead of `APPLIED`. Alert. Require manual acknowledgement to apply.
- If a fetched snapshot has 0 SKUs but previous had >0, always quarantine — never silently zero out a vendor.
- Thresholds are per-vendor config (`drop_ratio_threshold`, default 0.5), because some vendors legitimately oscillate.

### 4.6 Overlap & concurrency

- **REST/SFTP same-vendor:** Postgres advisory lock per `(source_type, vendor_id)`. If lock unavailable, run is skipped (logged, counted; not failed). Beat re-fires next tick.
- **Cross-vendor:** unlimited parallelism; vendors are independent.
- **Normalize tasks:** per `vendor_run` row, idempotency by `run_id` and `status` check (see §4.3 step 1). Two workers picking up the same SQS message both see `status = PENDING` initially, but a `SELECT ... FOR UPDATE SKIP LOCKED` on the run row serializes them.
- **DST handling:** Celery Beat is configured in UTC. All `cadence_cron` expressions are UTC. We do **not** schedule in vendor-local time; vendors with "9am their time" requirements are encoded as the UTC equivalent and noted in config. This means we do not get the DST double-fire or skip problems.

### 4.7 Backpressure / large delta

- Per-vendor full-snapshot size: ~10K–20K SKUs typical, ~50K worst case (Vendor B). 140K total across 12 vendors.
- Memory budget per normalize task: 512MB. Snapshots streamed via `csv.DictReader` for SFTP/email; REST pagination already chunks. A single normalize task does not need to hold the full diff in memory; we can stream-diff against the previous snapshot loaded as a hash table (50K rows × ~200 bytes = ~10MB, fits trivially).
- If catalog API is slow (e.g. p99 > 1s) and a vendor has 50K diffs, applying serially takes >50s. With 8-way concurrency, ~7s. Acceptable. If the diff is >100K (post-outage recovery), we chunk into 5K-SKU batches and yield between batches to let other vendors progress; budgeted at 5 minutes per run before the next run is forced to skip (advisory lock won't release).

### 4.8 Audit log

Postgres table `inventory_audit`:
- `id bigserial`, `vendor_id`, `run_id`, `sku`, `old_qty`, `new_qty`, `source_timestamp`, `applied bool`, `applied_at timestamptz`
- Index on `(sku, applied_at desc)` for "what's the recent inventory history for SKU X" queries.
- Retention: 18 months online, then archived to S3 Glacier. (Open question §10 OQ-3.)

### 4.9 Data model summary

```
vendor_config(vendor_id pk, source_type, cadence_cron, rate_limit_per_hour,
              source_config jsonb, secret_ref, field_mapping jsonb,
              staleness_threshold_minutes, drop_ratio_threshold, enabled,
              shadow_mode, created_at, updated_at)

vendor_run(run_id pk uuid, vendor_id fk, source_type,
           status enum(PENDING, RAW_LANDED, VALIDATION_FAILED, QUARANTINED,
                       APPLIED, FAILED, EMPTY),
           raw_s3_key, source_message_id unique nullable,
           started_at, finished_at, error_detail jsonb)
       index (vendor_id, started_at desc)
       unique (source_message_id) where source_message_id is not null

vendor_cursor(vendor_id pk, cursor_value jsonb, updated_at)

vendor_snapshot(snapshot_id pk uuid, vendor_id fk, run_id fk,
                sku_count int, applied_at timestamptz,
                status enum(APPLIED, QUARANTINED))
            index (vendor_id, applied_at desc)

inventory_audit(id bigserial pk, vendor_id, run_id, sku, old_qty, new_qty,
                source_timestamp, applied bool, applied_at)
              index (sku, applied_at desc)
              index (run_id)

unknown_sku_log(id bigserial, vendor_id, vendor_sku_id, seen_at, run_id)
```

### 4.10 External dependencies — what happens if each is down

| Dependency | If down | Behavior |
|---|---|---|
| Vendor REST API (any of 8) | 5xx / timeout | Retry 3× w/ backoff; then FAILED; staleness alert fires at threshold. Catalog keeps last good values. |
| Vendor SFTP server (any of 3) | Conn refused | Same as REST: retry, then FAILED, staleness alert. |
| Vendor K email | No delivery | Staleness alert at 30h. Catalog keeps last good values. No automated retry mechanism — by definition. |
| Catalog REST API | 5xx / timeout | Per-SKU retry; if persistent, mark run PARTIALLY_APPLIED. Resume on next run (idempotent upsert). Page on-call. |
| AWS S3 | Down | Pulls fail at landing step; runs fail; staleness alert. Regional S3 outage is well outside our SLO. |
| Postgres (our DB) | Down | All tasks fail. Service is effectively offline. RTO < 30 min via Multi-AZ failover (existing platform pattern). |
| Redis (rate limiter) | Down | Rate limiter degrades to "no rate limit"; for Vendor F that risks 429s. Fall back to per-vendor in-process semaphore with the same hourly limit, as a worst-case. |
| AWS SES inbound | Down | Email vendor stops landing. Staleness alert. SES inbound is regional; we accept the SPOF (see §6 reliability). |

## 5. Alternatives considered

### A. Push to vendors (have them call us)
Rejected. We are the smaller party for 11 of 12 vendors; we have no leverage to make them implement a webhook. PM has already negotiated this with vendor K — the email is the negotiated answer.

### B. Single shared adapter that does everything (no S3 raw landing)
Rejected. Conflates "we couldn't fetch" with "we couldn't parse" with "we couldn't write", which is exactly the failure-mode confusion the PRD calls out. Raw landing in S3 also makes replay free: if normalization has a bug, we re-process the same raw blob without re-hitting the vendor.

### C. Airflow / Step Functions instead of Celery Beat
Rejected for v1. Org already runs Django + Celery; adding Airflow is a new operational surface (DB, scheduler, web UI, DAG deploys) for what is a small set of simple cron tasks. Revisit if (a) we exceed ~50 sync configurations or (b) we need cross-task dependencies, neither of which is true today.

### D. Kafka or EventBridge as the internal bus
Rejected. The fan-in is 12 vendors at 1/15min worst case — peak ~50 messages/min. SQS is trivially sufficient, simpler to operate, and idiomatic on AWS with Celery. Kafka's value (high throughput, ordered streams, replay) is not needed; its operational cost (broker, ZK/KRaft, rebalancing) is not warranted.

### E. Write directly to the catalog DB
Rejected. The catalog service owns its schema and has invariants (price × availability constraints, denormalized search indexes) we don't want to bypass. Going through the existing REST API keeps the contract clean and gives us their validation for free.

### F. Do nothing — keep the Google Sheet
Rejected on the basis of the $80K/month loss figure in the PRD. The Sheet workflow is the explicit thing we're replacing; PM has committed to a 2-week shadow rollout as the safety net.

## 6. Non-functional requirements

Walking `references/nfrs-checklist.md` in order.

### Scalability
- **Target.** 12 vendors at launch, growing to ~25 in 2 years. 140K SKUs; ~20K SKUs touched per day in steady state. Peak ~50 normalize tasks/min.
- **Mechanism.** Each adapter and normalize-task is stateless and horizontally scalable behind SQS. Celery worker pool sized 4 by default; auto-scales on `normalize-queue` depth via ECS service auto-scaling.
- **Bottleneck.** Catalog service write API. We do not own its capacity. Diff-only writes minimize call volume (avg ~5% of SKUs change daily). At 8-way concurrency per vendor we will not exceed ~96 in-flight catalog writes; if catalog team raises a concern, we add a service-wide token bucket.
- **Verification.** Load test with synthetic vendor producing 100K-row CSV; catalog write API stubbed at p99 = 500ms; confirm full run < 5 min.

### Reliability & availability
- **Target.** 99% of scheduled runs per vendor per week succeed (status ∈ {APPLIED, EMPTY}). Freshness SLO: **per-vendor data is at most `staleness_threshold_minutes` stale**, alerted when breached.
- **SPOFs.** (1) Our Postgres — Multi-AZ. (2) SES inbound in us-east-2 — accept; regional outage is a known limitation of email ingestion. (3) Catalog service — out of scope, owned by another team, has its own SLO.
- **Dependency failure handling.** Each external call has timeout + 3 retries with exponential backoff + 30% jitter. Auth failures do not retry. Per-vendor circuit breaker would be over-engineered for the call volume; revisit if a vendor's flakiness causes worker starvation.
- **Idempotency.** Catalog API call is a `PUT` keyed on SKU — natural idempotency. `vendor_run` and `inventory_audit` writes guarded by run-row status check + unique constraints.

### Resilience
- **Graceful degradation.** A single vendor failing does not block the other 11. Each runs on its own Beat schedule with its own lock. Catalog API soft-down → reads continue from cache; writes queue.
- **Bulkheads.** Separate Celery queues per source type (`rest-pull-queue`, `sftp-pull-queue`, `email-queue`, `normalize-queue`) so a slow SFTP can't starve email handling.
- **Timeouts.** Every external call timed out (REST 30+60s, SFTP per-op 60s, catalog 5s, SES handler Lambda 60s).
- **Backoff & jitter.** Exponential with 30% jitter on all retries.

### Performance
- **Latency targets.** Not user-facing. End-to-end freshness (vendor data ready → catalog updated) target p95 < 30 minutes for REST vendors, < 2h for SFTP vendors, < 2h after email arrival for vendor K.
- **Resource budget.** 512MB memory per normalize task; CPU < 0.5 vCPU. Connection pool to catalog API: 8.
- **N+1 hazards.** SKU resolution is the obvious one. Mitigation: bulk lookup in chunks of 1000.

### Observability
- **Metrics** (CloudWatch / Datadog — match org standard):
  - `vendor_sync.runs.total{vendor_id, source_type, status}`
  - `vendor_sync.run_duration_seconds{vendor_id, source_type}` (histogram)
  - `vendor_sync.records_processed{vendor_id}` (counter)
  - `vendor_sync.diff_size{vendor_id, change_type=added|removed|changed}` (gauge)
  - `vendor_sync.staleness_seconds{vendor_id}` (gauge — derived from last successful run timestamp)
  - `vendor_sync.catalog_write.latency_ms` (histogram, by status code)
  - `vendor_sync.quarantine.total{vendor_id, reason}`
  - `vendor_sync.unknown_sku.total{vendor_id}`
- **Logs.** Structured JSON. Every log line carries `run_id` and `vendor_id`. PII not applicable (no end-user data).
- **Traces.** OpenTelemetry. Trace context propagated from Beat → adapter → SQS message → normalize task → catalog API call. Sampling 10%.
- **Alerts** (symptom-based, not cause-based):
  - **Page (P1):** `staleness_seconds > staleness_threshold_minutes * 60` for any vendor for 15+ minutes. This is the failure the PRD specifically wants caught.
  - **Page (P1):** any vendor auth failure (401/403).
  - **Page (P2):** N=3 consecutive `FAILED` runs for the same vendor.
  - **Ticket (P3):** any `QUARANTINED` run.
  - **Ticket (P3):** unknown_sku rate > 5% for a vendor on a given run.
- **Runbook.** `<link to runbook — to be added>`. Must cover: credential rotation per vendor, cursor reset, quarantine release, replaying a raw S3 blob, vendor K email troubleshooting.

### Security
- **AuthN to vendor REST APIs.** Vendor-specific (Basic, Bearer, OAuth2 client-credentials). Credentials in AWS Secrets Manager, one secret per vendor, IAM-scoped to the worker role.
- **SFTP auth.** SSH keys in Secrets Manager. Host-key pinning required — `paramiko`'s `set_missing_host_key_policy(RejectPolicy)`. **Mandatory.**
- **Email auth.** Sender-address verification via SES receipt + explicit allow-list per vendor. Note: this is weaker than DKIM/SPF enforcement; flagged as risk R-EMAIL-2.
- **AuthN to catalog API.** Service-to-service mTLS or IAM role (whichever catalog team supports today). Confirm with catalog owners — see §10 OQ-4.
- **Secrets rotation.** Manual on PRD timeline; we expose `python manage.py rotate_vendor_secret <vendor_id>` to ops. Automation is a follow-up.
- **Data classification.** No PII. Inventory quantities are commercially sensitive (treat as internal-confidential): no logging of secrets, no exposing raw vendor payloads outside the worker VPC. S3 bucket private, SSE-S3, no public access.
- **Input validation.** Per-vendor Pydantic schema on every payload. Vendor-supplied SKU IDs treated as untrusted strings (no SQL injection risk; we use ORM/parameterized queries).
- **Threat model.** Internal VPC + inbound email. Inbound email is the only path from the public internet; SES does coarse filtering. Attachment is parsed as CSV only — we do not execute or render it.
- **Supply chain.** Lockfile (`pip-compile` / `uv.lock`). Image signed via Notary in the existing org build pipeline.

### Cost
- **Estimated unit cost.** Negligible. ~100 Celery task executions/hour × 12 vendors, ~5GB S3/month for raw blobs, SES inbound ~$0.10/1000 messages. Catalog API calls are free (internal). Total ballpark < $200/month at launch.
- **Cost drivers.** S3 raw retention is the largest. Retention policy = 90 days → Glacier; documented in IaC.
- **Cost ceiling.** No alert needed at this volume. Revisit if S3 bill exceeds $500/month.

### Compliance & privacy
- **Residency.** N/A — no end-user PII. Inventory data has no residency requirement.
- **Retention.** Raw vendor payloads in S3 for 90 days → Glacier 18 months → delete. Audit log 18 months online → Glacier indefinitely.
- **Regulatory.** None applicable.
- **Audit log.** Yes — `inventory_audit` table, see §4.8. Covers PRD requirement explicitly.

### Deployment & operations
- **Deploy strategy.** Rolling deploy of Celery workers via existing ECS pattern. Beat schedule is one shard (single Beat process) — Beat HA is handled by existing org pattern (DB-backed scheduler with leader election).
- **Rollback.** Code rollback is trivial. Schema migrations are forward-only but reversible for v1 (no destructive changes); each migration ships with a documented down path.
- **Feature flags.**
  - `VENDOR_SYNC_GLOBAL_KILLSWITCH` — disables all scheduled tasks instantly.
  - Per-vendor `vendor_config.shadow_mode` — toggles apply on / off without redeploy.
  - Per-vendor `vendor_config.enabled` — pause a single vendor without redeploy.
- **Config.** `vendor_config` in Postgres, edited via Django admin (audited). Secret refs in Secrets Manager.
- **Migrations.** Django migrations; standard CI gate.

### Disaster recovery
- **RTO.** 4h for full service. RTO is dominated by Postgres failover and Celery cluster restart.
- **RPO.** ≤ 1h for our DB (Multi-AZ + PITR). Raw payloads in S3 are durable (99.999999999%). Cursor loss → replay from raw S3 blobs of the last 24h.
- **Backups.** Postgres PITR via RDS. S3 versioning on the raw bucket.
- **Multi-region.** No. Single region (us-east-2) per PRD. A regional outage is the same as "everything is down" — acceptable for an internal sync that already tolerates 24h staleness.

### Maintainability
- **Test strategy.**
  - Unit tests for each adapter and the normalize pipeline (Django test runner).
  - Per-vendor "golden fixture" test: a saved raw payload → asserted normalized snapshot.
  - Integration test against `moto` for S3/SES/SQS and a Postgres test container.
  - Contract test against the catalog API using a recorded fixture.
  - Manual smoke test in staging against a single real vendor before shadow-mode launch.
  - No chaos testing for v1.
- **Documentation.** This TDD; per-vendor mapping docs auto-generated from `vendor_config.field_mapping`; runbook (see Observability).
- **Ownership.** Supply Chain Engineering team (current owners of the catalog write path). On-call shared with catalog service.

### Extensibility
- **Variation points.** Adding a new vendor of an existing source type = config + Pydantic schema + field-mapping JSON. No code path changes. Adding a new source type = new adapter implementing the `Adapter` interface (3 methods: `discover_runs()`, `fetch_to_s3(run)`, `name`).
- **Versioning.** Internal contracts. `VendorInventorySnapshot` shape is versioned in code (`snapshot_schema_version`) and persisted on the row so we can re-normalize old raw blobs after a schema change.

## 7. Risks & failure modes

| ID | Risk | Likelihood | Blast radius | Mitigation | Recovery |
|----|------|-----------|--------------|-----------|----------|
| R-1 | Vendor F rate-limit (60/hr) exceeded → 429 storm | Medium | Vendor F goes stale | Token bucket per vendor in Redis; cadence + bucket sized below limit | Backoff on 429; staleness alert at 24h |
| R-2 | "Vendor briefly returns empty" → catalog shows 0 stock for everything | Medium | Whole vendor's inventory effectively offline; storefront cancels orders | Quarantine rule §4.5 (drop-ratio threshold) | Manual quarantine release after human confirms vendor state |
| R-3 | Cursor advances before successful raw landing → lost window | Low | Lost data for 1 cycle | Cursor advance only after S3 write succeeds (§4.4) | Manual cursor rewind from runbook |
| R-4 | Two Beat processes both fire = duplicate runs | Low | Wasted API calls; possible 429 | Postgres advisory lock per `(source_type, vendor_id)` | Self-healing — second run skips on lock fail |
| R-EMAIL-1 | Vendor K's email arrives malformed (attachment missing, wrong CSV shape) | Medium-High | Vendor K stale | Validation fails → run marked `VALIDATION_FAILED` → P3 ticket + staleness alert at 30h | Ops contacts vendor K; replay raw email blob once corrected attachment received |
| R-EMAIL-2 | Spoofed email from Vendor K's address | Low | Catalog corrupted for Vendor K's SKUs | Sender allow-list + (proposed) DKIM enforcement (§10 OQ-5) | Quarantine rule §4.5 catches mass deletions; audit log lets us replay from previous good snapshot |
| R-5 | Catalog write API rate-limits us | Medium | Slow apply; possible PARTIAL_APPLIED runs | Concurrency limit (8); on 429 back off and queue | Resume on next run (idempotent) |
| R-6 | Unknown SKU explosion (vendor renamed half their catalog) | Low | Lots of `unknown_sku_log` entries; no catalog update for those SKUs | Alert at 5% threshold | Ops investigates, updates SKU mapping |
| R-7 | Shadow-mode disagreements not actionable | Medium | 2-week shadow-mode is wasted | Build a "shadow diff" report tool from `inventory_audit.applied=false` rows | PM + Supply Chain ops triage daily during shadow window |
| R-8 | Secrets rotation expires mid-window → silent auth failure | Medium | Vendor stale | Auth failure pages immediately (no silent retries) | Ops rotates and re-enables |
| R-9 | DST footgun in vendor-supplied timestamps | Low-Medium | Diff against wrong window | All cursor math in UTC; reject non-UTC timestamps without explicit timezone | Manual cursor reset |

## 8. Rollout & migration

**Phasing.**

1. **Week 1–2:** infrastructure (DB, S3 buckets, SES rule, IaC), config model, one REST adapter against one real vendor in shadow mode. Define `VendorInventorySnapshot`.
2. **Week 3:** remaining REST vendors (7 more); SFTP adapter against one real vendor.
3. **Week 4:** remaining SFTP vendors (2 more); email adapter (Vendor K).
4. **Week 5:** Shadow mode for all 12 vendors. Daily review of `inventory_audit` diffs vs Google Sheet workflow.
5. **Week 6:** Per-vendor cutover. Flip `shadow_mode = false` one vendor at a time, starting with the cleanest REST vendors and ending with Vendor K.

**Backwards compatibility.** N/A — net-new system. Google Sheet workflow continues in parallel until per-vendor cutover.

**Rollback.**
- Code: standard ECS rollback.
- Behavior: flip `shadow_mode = true` on the affected vendor — instant.
- Data: catalog writes are upserts. There is no "undo a write" — the previous-value record is in `inventory_audit`. A bulk-rollback path is `python manage.py revert_vendor_to_snapshot <vendor_id> <snapshot_id>`, which replays a prior `vendor_snapshot` as a forced update.

## 9. Observability & operations

Already covered in §6 Observability. Operational surface, summarized:

- **On-call surface area.** Supply Chain Eng on-call gets:
  1. Staleness alerts per vendor.
  2. Auth failure alerts.
  3. Consecutive-failure alerts.
  4. Quarantine tickets.
- **Routine ops actions** (runbook items): rotate vendor secret; release quarantined run; rewind cursor; replay raw blob; pause/resume vendor; force-resync.
- **Dashboards.** One per-vendor health row (status, last success, current staleness, run count today, error count today). One service-wide overview.

## 10. Open questions

1. **OQ-1: Where does `vendor_config` live?** Postgres table (editable in admin) vs YAML in repo (audited via PR). *Recommendation:* Postgres for credentials/cadence/flags (operational), YAML/code for field mappings and Pydantic schemas (engineering). Hybrid.
2. **OQ-2: Discontinued-SKU pipeline to storefront** (PRD open question). *Recommendation:* publish a `sku.discontinued` domain event to the existing event bus on `removed` diffs that persist for N consecutive runs (N=3). Out of scope for v1 unless storefront team confirms they consume it.
3. **OQ-3: Audit log retention** — 18 months online is a guess. Confirm with finance/legal whether SOX or similar applies to inventory history. *Assumption:* it does not; 18 months is generous.
4. **OQ-4: Catalog API auth mechanism** — mTLS or IAM-signed? Need to confirm with catalog team before week 1. *Assumption:* IAM (standard internal pattern).
5. **OQ-5: DKIM / SPF enforcement for Vendor K email.** Should we require DKIM-signed messages? Vendor K is small and probably does not DKIM-sign. *Recommendation:* combine sender-allow-list + quarantine-on-large-drop (§4.5); revisit if a security review demands stricter.
6. **OQ-6: Shadow-mode arbitration** (PRD open question). When new system disagrees with the Sheet, who decides? *Recommendation:* Supply Chain ops lead, with a 24h SLA for arbitration during the shadow window. Engineering produces the diff; ops owns the call.
7. **OQ-7: Beat HA.** Single Beat process is a SPOF for *scheduling*. The org may already have a DB-backed Celery Beat with leader election; if not, accept a 1-tick gap on Beat restart. *Assumption:* org standard pattern exists; confirm in week 1.
8. **OQ-8: Backfill semantics for first run.** For vendors whose APIs support "since" cursors, what's the right initial lookback? 24h, 7d, or full-snapshot? *Recommendation:* full-snapshot on first run, then `since=last_run` thereafter — simplest, eliminates an edge case.
9. **OQ-9: Per-vendor cadence values.** Need actual numbers from each vendor's docs / rate limits. *Assumption while drafting:* REST = every 15 min except Vendor F (every 1h to stay under quota); SFTP = nightly at 02:00 UTC + 30 min for each vendor; Vendor K = event-driven (no schedule).
