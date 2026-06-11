### Land the 1 GB throughput benchmark and synthetic fixture

**One-liner:** Build the synthetic 1 GB JSON-lines fixture and the CI benchmark that asserts the §6 throughput NFR (1 GB / 60s for a streaming `SELECT WHERE LIMIT`), and gate releases on it.

**Composes:**
- A reproducible generator for a synthetic ~1 GB JSON-lines fixture with a realistic schema (timestamp, level, message, a handful of structured fields), seeded for determinism so runs are comparable.
- A CI benchmark job that runs the fixture through a streaming `SELECT ... WHERE ... LIMIT ...` query and asserts wall-clock under 60 seconds on the team's reference CI runner, with the runner spec named in the benchmark output.
- Headroom-aware threshold: the assertion includes a documented margin so transient CI noise doesn't flap the build, but a real regression (say, 25%+) trips it.
- Output that names which operator and which input size produced the measured throughput, so a regression report points at a real cause.
- This is **only** the throughput benchmark — the cold-start benchmark belongs to task 01 and stays there.

**TDD sections addressed:** §6 NFR throughput (1 GB / 60s), §3 High-level approach (streaming pipeline is the load-bearing claim being verified here).

**Depends on:** Walking skeleton: streaming SELECT/WHERE/LIMIT with CSV output, Implement schema-on-read coercion, `--strict-schema`, and malformed-line handling

**Acceptance criteria:**
- The fixture generator produces a ~1 GB JSON-lines file deterministically from a seed; running it twice yields byte-identical output.
- A streaming `SELECT WHERE LIMIT` query over the fixture finishes within 60 seconds on the documented CI runner spec, with the measurement recorded in the build log.
- The benchmark fails the build when throughput regresses past the documented headroom threshold.
- The benchmark is reproducible locally — a developer can run it on their laptop and get a number that's apples-to-apples with CI (with the laptop hardware noted).
- The fixture is either committed to the repo via Git LFS or generated on-demand in CI — whichever is chosen is documented, with the size-vs-checkout-time tradeoff named.
- Memory usage during the benchmark stays well under the default 512 MB budget, confirming the streaming operators actually stream end-to-end.
