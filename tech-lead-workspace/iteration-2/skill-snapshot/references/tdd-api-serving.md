# TDD reference — API serving (synchronous request/response)

## When this fits

A long-running service that accepts synchronous requests and returns
responses — REST, GraphQL, gRPC, RPC, BFF, internal microservice. Callers
block on the response (within a budget). Examples: a billing API, a search
endpoint, a user-profile service, an admin dashboard backend.

**Not this** if: the system primarily consumes from a queue/topic (see
event-driven / async worker), is push-from-third-party (see webhook
receiver), or is a one-shot invocation (see CLI).

## What's different about APIs

- **Latency budget is real.** The caller is waiting. Every DB query, every
  external call, every lock counts.
- **Backwards compatibility is contractual.** Once a client depends on the
  API, breaking changes have a cost. Versioning is a first-class concern.
- **Concurrency is constant.** Multiple in-flight requests means
  per-request state, connection pooling, and the usual concurrency hazards.
- **You are the dependency.** Other systems will retry you, fan out to
  you, and complain when you're slow. Their failure modes become yours.

## Required sections (in addition to the common skeleton)

### 4a. API surface

- Protocol (REST / GraphQL / gRPC) and why.
- Resource model / RPC list. For REST: routes, methods, status codes,
  request/response schemas. For GraphQL: schema sketch. For gRPC: proto.
- Versioning strategy — URL prefix (`/v1/...`), header (`Accept-Version`),
  or proto package version. How breaking changes are signaled and
  rolled out.
- Idempotency strategy on writes — idempotency keys, conditional ETags,
  or natural (e.g. PUT on an ID).

### 4b. AuthN / AuthZ

- Caller identity — JWT, mTLS, API key, signed cookie, IAM role.
- Where the token is validated (gateway? handler? both?).
- Authorization model — RBAC, ABAC, row-level. Where policy is enforced.
- Service-to-service vs user-facing — they're different threat models.

### 4c. Data access

- Database(s) used and why (Postgres, MySQL, DynamoDB, Mongo, …).
- Schema sketch with primary keys and indexes for the query patterns.
- Per-request query budget (e.g. "≤ 3 queries per GET, no N+1").
- Caching layer (Redis, in-process, CDN) — where, what TTL, how
  invalidated, what staleness is acceptable.
- Connection pool sizing relative to expected concurrency.

### 4d. Latency budget

Build a budget table:

| Stage              | Target (p95) | Mechanism                          |
|--------------------|--------------|-------------------------------------|
| TLS + ingress      | 5ms          | shared LB                           |
| Auth validation    | 5ms          | cached JWKS                         |
| DB queries (×N)    | 30ms total   | indexed primary-key lookups         |
| External call(s)   | 50ms each    | timeout 80ms, 1 retry with jitter   |
| Serialization      | 5ms          | JSON, small payloads                |
| **End-to-end**     | **≤ 150ms** |                                     |

Show the math. If the budget doesn't fit, the design needs to change.

### 4e. Concurrency, rate limits, and back-pressure

- Per-instance concurrency (worker / thread / event-loop model).
- Per-tenant or per-user rate limits — token bucket, leaky bucket, fixed
  window. Where enforced.
- Behavior at limit — 429 with `Retry-After`, queue, drop. State the choice.
- Connection draining on shutdown / deploy.

### 4f. Error contract

- Error response shape (RFC 7807 Problem Details, or custom — be consistent).
- HTTP status mapping — what's a 400 vs 422 vs 409 vs 500.
- Which errors are retryable by the client (and signal that via status /
  header).
- Correlation / trace ID returned in every response.

## NFRs that matter for APIs

From `nfrs-checklist.md`, the high-signal ones for an API are:

- **Availability SLO** — be specific (99.9? 99.95?). Drives redundancy
  and deployment strategy.
- **Latency p50/p95/p99** — see budget above.
- **Capacity / scalability** — RPS target, scaling axis, stateful bottleneck.
- **Observability** — RED metrics per route, structured logs with trace IDs,
  trace propagation upstream and downstream.
- **Security** — full threat model since this is usually internet-adjacent or
  cross-tenant.
- **Deployment** — zero-downtime story, canary, rollback.

Often omitted (justify the omission): multi-region (unless real DR need),
deep cost modeling (unless very high volume).

## Common pitfalls

- **No timeout on downstream calls** — one slow dependency saturates the pool.
- **Retries without budgets** — amplifies upstream failures into outages.
- **N+1 queries** — fine in dev, melts the DB in prod.
- **Cache without invalidation story** — stale reads, inconsistent state.
- **Status codes that mean nothing** — every error returns 500, clients
  can't tell what to retry.
- **AuthZ in handlers, AuthN in gateway** — a missed handler ships an
  unprotected endpoint.
- **Schema migrations that block writes** — column adds with defaults,
  index creation, unique constraint backfills.
- **Treating "internal" as "trusted"** — service-to-service still needs
  authn, authz, input validation.

## Key decisions to surface in "Alternatives considered"

- Protocol choice (REST vs gRPC vs GraphQL).
- DB choice — and the sharding / replication story.
- Sync vs async — could this be a queue + worker instead?
- Whether to put a cache in front (and what consistency window is acceptable).
- Versioning strategy and how v2 will be rolled out alongside v1.
