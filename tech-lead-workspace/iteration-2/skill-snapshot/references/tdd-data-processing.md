# TDD reference — data processing (batch / ETL)

## When this fits

A pipeline that reads a bounded dataset, transforms it, writes results.
Usually long-running, throughput-oriented rather than latency-oriented.
Examples: nightly ETL, ad-hoc backfills, dataset rebuild, dbt-style
transformations, data-warehouse loads, report generation, ML feature
pipelines run on a schedule.

**Not this** if: the data is unbounded / streaming (see event-driven), the
trigger is per-message (see async worker), or the trigger is a clock without
heavy data work (see scheduled pull).

## What's different about batch pipelines

- **Throughput, not latency.** Optimize for total wall-clock and cost per
  GB / per row, not per-record latency.
- **Reproducibility matters.** A failed run must be re-runnable safely. Most
  of the design effort goes into making that possible.
- **Side effects are the danger zone.** Writes to external systems
  (notifications, downstream tables, third-party APIs) must be designed for
  re-runs.
- **Failure mid-run is the common case.** Plan for restart from checkpoint,
  not "we'll just rerun from scratch" — for big jobs that's prohibitive.

## Required sections (in addition to the common skeleton)

### 4a. Data sources & sinks

- Source(s) — system, format (Parquet, CSV, JSON, JDBC table), location,
  approximate volume.
- Sink(s) — system, format, partitioning, how it's read by downstream
  consumers.
- Schema for the input (expected, including bad-data tolerance) and the
  output (committed contract).

### 4b. Execution model

- Engine — Spark, Airflow-orchestrated Python, dbt, Beam, plain SQL on
  warehouse, custom. Why this one.
- Parallelism unit — partition, file, shard, row group.
- Distribution — single node, single cluster, distributed. Where the
  shuffles / joins happen and how they're sized.
- Resource sizing — workers / executors, memory per worker, expected
  runtime.

### 4c. Transformation logic

- Step-by-step description of the transformations, ideally as a DAG.
- Joins, aggregations, deduplication strategy.
- Late-arriving data handling — does this job assume the source is
  complete? How is "completeness" determined?
- Data quality checks — column nullability, value ranges, row counts vs
  expected. Where they run and what they do on failure (warn? fail? quarantine?).

### 4d. Idempotency & re-runs

- **What guarantees re-runs produce the same result?** Deterministic input
  partitioning, no side effects on non-final tables, no "current time" in
  the transformation (use logical run date).
- **What does partial re-run mean?** Per-partition? Whole job?
- **How are outputs committed?** Atomic swap, write-to-staging-then-rename,
  partition replace, transactional table (Iceberg / Delta / Hudi).
- **What if the job is killed mid-write?** No half-written partitions
  visible to downstream readers.

### 4e. Failure handling

- What types of failures are expected (transient — network blip, infra
  preemption; permanent — bad data, schema mismatch)?
- Per-record vs per-partition vs whole-job failure. When do we skip / quarantine
  vs fail the run?
- Retry policy at the orchestrator (how many, with backoff).
- Dead-letter location for records that can't be processed.

### 4f. Lineage & data quality

- How is provenance recorded — which run produced which rows.
- Downstream consumers — what they expect, how they know data is ready
  (success file? table-version metadata? Airflow sensor?).

### 4g. Cost & runtime

- Expected per-run cost and runtime (range, not a single point).
- What knobs change cost vs runtime (parallelism, instance type).
- Cost ceiling / budget alert.

## NFRs that matter for batch

From `nfrs-checklist.md`:

- **Performance** — throughput (rows/sec or GB/hour), not latency. Job
  duration target.
- **Cost** — directly proportional to data volume; high-signal.
- **Reliability** — successful runs per N attempts. SLA for downstream
  data freshness ("yesterday's data available by 06:00 UTC").
- **Observability** — per-step duration, row counts, data-quality metric
  trends. Alerts on freshness SLO breach.
- **Maintainability** — test strategy (especially for data correctness:
  golden datasets, snapshot tests).
- **Compliance** — what data lands where, retention.

Often omitted: availability SLO (it's a job, not a service), low-latency
performance (irrelevant), real-time monitoring (alert on freshness
breach is enough).

## Common pitfalls

- **Non-idempotent side effects.** Sending an email per row, then a
  re-run sends them all again.
- **Implicit "now()" in the transformation.** Results differ between runs
  for the same logical date. Always use the orchestrator-injected run date.
- **Whole-job-fail on one bad row.** No quarantine, no partial progress —
  every restart starts from zero on a multi-hour job.
- **Schema drift not detected.** Source quietly adds a column; downstream
  contract breaks weeks later.
- **Joins that explode** when one side has unexpected duplicates.
- **No partition pruning** — full-table scans on every run.
- **Downstream readers race the writer** — no atomic commit, readers see
  half-written state.

## Key decisions to surface in "Alternatives considered"

- Engine choice (Spark vs SQL-on-warehouse vs Python vs Beam).
- Storage format (Parquet + Hive partitions vs Iceberg / Delta vs warehouse
  native).
- Orchestrator (Airflow / Dagster / Prefect / cron + lock file).
- Full refresh vs incremental — and how late-arriving data is handled in
  the incremental case.
- Where the data quality gate lives (in-job vs separate validation job).
