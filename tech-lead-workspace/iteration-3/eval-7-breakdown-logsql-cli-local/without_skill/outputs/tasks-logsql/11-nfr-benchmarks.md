# 11 — NFR benchmarks in CI

## Outcome

Two CI-enforced benchmarks that gate releases: cold-start first-row latency
and streaming throughput. Failure of either fails the build.

## Why

TDD §6 names both as "verified" NFRs. Without enforcement they will rot.

## Acceptance criteria

- **Benchmark A — cold start.**
  - Synthetic 10k-row JSON-lines file.
  - Query: `SELECT * WHERE level = 'error' LIMIT 10`.
  - Measure: wall-clock time from process start to first row on STDOUT.
  - Target: **< 100 ms** on the CI runner reference machine.
  - CI fails if median of 5 runs exceeds the target with a 10% margin.
- **Benchmark B — throughput.**
  - Synthetic 1 GB JSON-lines file generator checked into the repo (or
    generated on the fly in CI from a fixed seed).
  - Query: `SELECT ts, msg WHERE level = 'error' LIMIT 1000000`.
  - Target: **>= 1 GB / 60 s** sustained.
  - CI fails if throughput drops below 1 GB / 75 s (15% margin).
- Both benchmarks have a documented "reference machine" profile and a
  documented procedure for re-baselining when the runner changes.
- A `make bench` target runs both locally and prints a comparison against
  the last committed baseline.

## Out of scope

- GROUP BY / ORDER BY benchmarks — fine to add in v1.1 if needed.
- Per-PR benchmark gating with statistical significance — overkill for v1;
  the median + margin check is enough.

## Dependencies

05, 06, 07, 08, 09 (needs a working end-to-end binary).
