# 12 — Distribution: Homebrew + internal repo

## Outcome

`logsql` v0.1 installable via two paths: a Homebrew formula and a direct
binary download from the internal package repo. v0.2 broadcast plan
documented.

## Why

TDD §8: Homebrew + internal binary download are the named distribution
channels; v0.1 goes to platform-team volunteers for one week before broad v0.2.

## Acceptance criteria

- **Homebrew formula** in a tap repo:
  - Pins to a release tag.
  - Installs the static binary for macOS arm64 and amd64.
  - `brew install logsql && logsql --version` works.
- **Internal repo:**
  - Release pipeline uploads the linux-amd64, linux-arm64, darwin-amd64,
    darwin-arm64 binaries to the internal package repo with checksums.
  - Documented one-liner curl install.
- **Versioning:**
  - Git tags drive the version string from task 01.
  - `CHANGELOG.md` populated for v0.1.
- **v0.1 → v0.2 plan documented:**
  - Volunteer cohort identified (platform team).
  - Feedback channel named.
  - Exit criteria for "freeze the documented SQL subset" (TDD §8) recorded.
- **Rollback:** `brew uninstall logsql` documented; older binaries kept in
  the internal repo for at least 90 days.

## Out of scope

- Linux distro packages (apt, rpm) — defer until demand is real.
- Windows — out of scope; TDD does not name it as a target platform.

## Dependencies

01, 11. All earlier tasks must ship for there to be a binary to distribute.
