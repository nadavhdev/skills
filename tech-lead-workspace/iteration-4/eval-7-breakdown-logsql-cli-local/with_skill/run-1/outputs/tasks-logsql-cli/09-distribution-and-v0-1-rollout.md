### Ship distribution: Homebrew formula, internal binary repo, v0.1 rollout

**One-liner:** Make `logsql` installable on engineer laptops via Homebrew and direct binary download from the internal repo, and run the §8 staged rollout (platform-team volunteers on v0.1, broad publish on v0.2).

**Composes:**
- A Homebrew formula that installs the statically-linked binary for macOS (both architectures the team uses) and Linux, pinned to a release tag.
- Direct binary download from the internal package repo as a parallel install path, for engineers who don't use Homebrew.
- A release pipeline that produces signed, statically-linked binaries per supported platform and publishes both the Homebrew bottle and the internal-repo artifacts from the same tagged commit.
- v0.1 release to platform-team volunteers for one week with a feedback channel; v0.2 broad publish only after the documented SQL subset is frozen — gating the v0.2 publish on the §10 open question being resolved.
- A documented rollback path: `brew uninstall logsql` plus instructions for pinning to a prior internal-repo binary, matching §8.
- README / install docs at release time covering the command surface, exit-code convention (especially the unusual `1` = zero rows), and the documented SQL subset.

**TDD sections addressed:** §3 High-level approach (static binary distribution requirement), §8 Rollout & migration.

**Depends on:** Walking skeleton: streaming SELECT/WHERE/LIMIT with CSV output, Implement the strict recursive-descent SQL parser, Implement GROUP BY and aggregate functions with memory budget enforcement, Implement ORDER BY with memory budget enforcement, Implement schema-on-read coercion, `--strict-schema`, and malformed-line handling, Add JSON-lines output format (`-o json`), Implement diagnostics: STDERR format, `--explain`, and panic handler, Land the 1 GB throughput benchmark and synthetic fixture

**Acceptance criteria:**
- `brew install <tap>/logsql` on macOS and Linux installs the binary, which then runs the README's example queries successfully end-to-end.
- A fresh laptop with no Go runtime can run `logsql` from the installed Homebrew binary — confirming the static-link requirement from §3.
- The internal-repo binary download path produces a binary with the same SHA as the Homebrew bottle for the same release tag.
- v0.1 is shipped to the platform-team volunteer list with a written feedback channel; v0.2 is held until the §10 SQL subset question is resolved.
- The release notes document the exit-code convention (notably `1` for zero rows matched) prominently, because it differs from common CLI expectations.
- Rollback is verified: a user can `brew uninstall logsql` and install a prior version from the internal repo, both documented step-by-step.
