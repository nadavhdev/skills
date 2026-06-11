# 01 — Project scaffold and build

## Outcome

A Go module skeleton that produces a statically-linked `logsql` binary via a
single build command, with CI running unit tests and lint on every push.

## Why

Static binary distribution is a hard requirement (TDD §3, §8). Every later task
depends on this baseline existing so engineers can run `go test ./...` and
produce a runnable binary locally.

## Acceptance criteria

- `go.mod` declares the module; Go version pinned to a current stable release.
- `cmd/logsql/main.go` exists and prints a stub `logsql vX.Y.Z` when run.
- `make build` (or equivalent) produces a static binary; verified with
  `file logsql` showing "statically linked" on Linux.
- CI pipeline runs: `go vet`, `go test ./...`, `golangci-lint run` on every PR.
- Version string is set at build time via `-ldflags` from a `VERSION` file or
  git tag.
- `README.md` documents the build and test commands.

## Out of scope

- Distribution artifacts (Homebrew formula, internal repo upload) — task 12.
- Any actual SQL/parsing/execution code.

## Dependencies

None. Unblocks all other tasks.
