# TDD reference — library / internal SDK

## When this fits

Code that other services consume; no runtime of its own. Examples: an
internal "auth client" library, a generated API SDK, a shared data-access
layer, a feature-flag client, a metrics wrapper, a domain-event-publishing
helper. Lives in a repo, ships as a package (pip, npm, Maven, Go module,
crates.io, etc.).

**Not this** if: the code runs as its own process (any of the other
categories).

## What's different about libraries

- **The runtime is someone else's process.** Latency, observability, and
  resource use are charged to the host, not to you. Behaviors that would
  be fine in your own service can ruin a host (hot loops, blocking I/O on
  module load, swallowing exceptions, hidden threads).
- **API stability is the product.** Breaking changes have a cost
  proportional to the number of consumers × their release cadences.
  Versioning is everything.
- **The library cannot crash the host.** Misuse should produce clear
  errors at the boundary, not a TypeError from line 1294 of a private
  module two hours into a job.
- **Configuration has to come from the host.** Don't read env vars or
  files behind the host's back; expose explicit configuration.
- **There are no logs / metrics until someone hooks them up.** Either
  accept a logger / metrics-registry from the host or document the
  callbacks / hooks.

## Required sections (in addition to the common skeleton)

### 4a. Public API surface

- The functions / classes / types that consumers will import. Annotated
  with stability (stable / experimental / internal).
- What's deliberately **not** exported (private modules, helpers,
  internal types).
- Naming conventions for symbols, modules, packages.
- Initialization model — global singleton, factory-returns-client,
  context-managed.

### 4b. Versioning & compatibility

- Versioning scheme — SemVer (almost always). Exact rules for what
  bumps major / minor / patch.
- Compatibility window — how many majors are supported in parallel.
- Deprecation policy — how a function is deprecated (warning, doc note,
  removal cadence).
- Migration story for breaking changes — codemods, deprecation shims,
  side-by-side imports.

### 4c. Dependency posture

- Minimum host language / runtime versions supported.
- Transitive dependencies — explicit list with rationale. Each dep is
  contagion risk for every consumer.
- Optional / extras — features that pull in heavy deps live behind
  optional install groups.
- Compatibility with common host frameworks (e.g. "Django >=4.2, Flask >=
  2.3, FastAPI >= 0.100").

### 4d. Configuration model

- Where config comes from — constructor args (preferred), explicit
  `configure()` call, builder pattern.
- **Avoid** reading from env vars / files automatically inside the
  library — surprise behavior the host can't see.
- Validation at config time — fail fast on bad config, not at first use.
- Sensitive config (secrets) — accept opaquely, never log.

### 4e. Error model

- Exception hierarchy / error types — derived from a single library-level
  base class so consumers can `except YourLibError`.
- Which errors are retryable by the caller; signal via type or attribute.
- Never swallow exceptions silently. Wrap and re-raise with context, not
  hide.
- No `print` statements; emit through a configurable logger.

### 4f. Concurrency posture

- Thread-safety statement — what's safe to share across threads,
  what's not.
- Async support — sync API, async API, or both (and which is canonical).
- Resource ownership — connection pools, background threads, file
  handles. Lifecycle (close / shutdown) explicit.
- No hidden background threads. If one is started, document it,
  expose a shutdown.

### 4g. Observability hooks

- Accept a logger from the host (or use the standard host-language
  logger by convention — `logging` in Python, `slog` in Go, etc.).
- Expose hook points for metrics / tracing without forcing a dependency
  on a specific lib (callback interface, or pluggable middleware).
- Document log level for each event class.

### 4h. Testing & release

- Test matrix — host language versions, host framework versions, OSes
  (where relevant).
- CI gates — lint, type-check, test, build, version bump enforcement.
- Release process — branch / tag / publish; who is allowed to publish;
  registry credentials handling.
- Changelog convention — auto-generated from commits or hand-written.

## NFRs that matter for libraries

From `nfrs-checklist.md`:

- **Maintainability** — public API stability, test coverage on the
  surface, doc quality.
- **Extensibility** — clear plugin / hook points; backwards-compatible
  evolution.
- **Security** — supply-chain (lockfile, signed releases, dep scanning),
  no secret leakage in error messages or logs.
- **Performance** — library overhead per call (named in ns/µs/ms, not
  "low"). Memory footprint when imported. Module-load side effects (none,
  ideally).
- **Compatibility** — host runtime / framework matrix.

Often omitted: availability (host's concern), DR (host's concern), real-
time metrics (host instruments the host).

## Common pitfalls

- **Side effects at import time** — opening connections, reading files,
  spawning threads. Breaks tests, breaks serverless cold-start budgets,
  surprises hosts.
- **Reading env vars / config files without the host knowing.**
- **Hidden background threads / timers** that prevent the host process
  from shutting down.
- **Catch-and-log instead of raise** — host has no idea something failed.
- **One mega-package with optional features** — every consumer pays the
  dep cost of features they don't use.
- **Breaking change in a minor version** because "no one uses that" —
  someone does, always.
- **Logging via `print`** — bypasses host log infrastructure.
- **Globals / singletons that prevent multiple configurations** in one
  host (e.g. multi-tenant test runners).

## Key decisions to surface in "Alternatives considered"

- Sync-only, async-only, or dual-API library.
- Single package vs split packages (core + optional integrations).
- Generated vs hand-written client (for API SDKs).
- Whether to expose low-level + high-level APIs or only one.
- Where to draw the "what belongs in the lib vs the host" line — e.g.
  retry policy in the lib (opinionated) vs surfaced as config (flexible).
