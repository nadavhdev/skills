# Architect review aspects — data processing (batch / ETL)

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-data-processing.md` first — that
is the standard this TDD was (or should have been) written against. Use its
required sections, NFR applicability, and pitfalls as the baseline checklist.
This file adds the architect lens.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **Output schema & partitioning.** Downstream consumers and re-processing both
  couple to the output layout. Reshaping it later means rewriting consumers.
- **Idempotency / re-run semantics.** Whether a failed or re-run batch produces
  the same result vs. duplicates is foundational and painful to bolt on later.
- **Storage format & location.** The chosen format (Parquet/CSV/table) and its
  compaction/retention story.

## Type-specific hot spots (review questions)
- **Re-run safety.** Can a partially-failed batch be safely re-run without
  double-counting or corrupting output? If not → `risk`/`data` finding. This is
  the single most-missed property in batch designs.
- **Bounded memory.** Joins, sorts, aggregations, de-dup — is the memory bound
  named and matched to data volume + growth, or will it OOM at scale
  (`scalability`)?
- **Partial-failure handling.** One bad record / one failed partition — does the
  job fail whole, skip-and-report, or quarantine? Is that deliberate?
- **Incremental vs full.** If incremental, is there a watermark/checkpoint that
  survives failure? Data-skew handling for hot keys?
- **Input schema evolution + data-quality checks.** Does the design validate
  inputs and handle upstream schema drift?

## Over-engineering & simplification tells
- Spark / a distributed cluster for data that fits comfortably on one node →
  point at the single-process shape (`over-engineering`/`simplification`).
- A streaming system for what the PRD describes as a nightly/periodic batch
  (wrong workload — that's scheduled-pull or event-driven).
- A bespoke orchestrator where an existing scheduler/DAG tool in the codebase
  already runs (`codebase-fit`).

## Calibration — NFRs
- **Demand if absent:** re-run/idempotency semantics, memory bound for the heavy
  operations, data-quality/validation, partial-failure behavior, a **per-run cost
  range + the dominant cost driver** (cost scales with data volume here), and a
  **data-correctness test strategy** (golden / snapshot datasets).
- **Flag as over-reach if present:** low-latency SLOs, high-availability serving
  guarantees — a batch job's contract is freshness + correctness, not uptime.
