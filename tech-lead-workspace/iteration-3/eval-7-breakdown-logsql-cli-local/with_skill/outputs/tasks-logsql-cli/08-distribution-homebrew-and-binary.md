### Distribution: Homebrew formula and internal binary download

**One-liner:** Ship `logsql` to engineers' laptops through a Homebrew formula and a direct binary download from the internal package repo, including the staged rollout from v0.1 (platform-team volunteers) to v0.2 (broad publish).

**Composes:**
- Build pipeline producing statically-linked binaries for the target laptop platforms (macOS arm64, macOS amd64, Linux amd64 at minimum), each verifiable as runnable without a Go runtime on the host.
- Release artifacts (binaries + SHA-256 checksums) published to the internal package repo at a stable URL pattern so `curl | install` instructions are durable across releases.
- Homebrew formula (tap or internal) that installs the correct architecture binary and verifies its checksum against the release manifest. `brew uninstall logsql` is the documented rollback path.
- Version string embedded at build time so `logsql --version` and the panic handler's issue-report block both report the published version.
- v0.1 release tagged and announced only to platform-team volunteers; v0.2 release tagged and announced broadly once the documented SQL subset is finalized (gated on the open question in TDD §10).
- README in the repo documents both install paths, the supported platforms, and the uninstall / rollback flow.

**TDD sections addressed:** §8 Rollout & migration (Homebrew, internal binary, staged release, rollback).

**Depends on:** Walking skeleton: SELECT / WHERE / LIMIT over JSON-lines to CSV

**Acceptance criteria:**
- `brew install <tap>/logsql` on a clean macOS arm64 laptop installs a runnable binary; `logsql --version` reports the released version.
- Direct-download install via the documented `curl` one-liner produces a binary whose SHA-256 matches the published checksum, and which runs on the target platform without an external runtime.
- `brew uninstall logsql` removes the binary cleanly; a subsequent install of an older published version restores the previous behavior.
- A binary produced for one architecture (e.g. macOS amd64) refuses to run on a different one with a clear OS-level error rather than a silent crash, so users picking the wrong artifact aren't confused.
- v0.1 is published to the internal package repo and announced only to platform-team volunteers; v0.2 publication is gated on the SQL-subset open question being resolved.
- Release notes for v0.1 and v0.2 list the documented SQL subset, the four exit codes, and the install / uninstall flow.
