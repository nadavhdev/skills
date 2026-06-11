# TDD reference — webhook receiver

## When this fits

An HTTP endpoint that receives **push notifications from a third party** —
Stripe events, GitHub webhooks, Twilio callbacks, payment processor
notifications, SaaS integration callbacks, partner integrations. The
sender is outside your trust boundary, may retry on its own schedule, and
you don't control the contract.

**Not this** if: the caller is your own client (see API serving), or the
flow is internal queue-to-worker (see async worker).

## What's different about webhook receivers

- **The sender's retry behavior is not yours to set.** Senders retry on
  their schedule (often aggressively), so you *will* receive duplicates and
  out-of-order events. Idempotency is not optional.
- **The endpoint is on the public internet.** Authenticity has to be
  cryptographically verified — IP allowlists alone are insufficient.
- **Latency to respond matters in a different way.** Most senders treat
  >X-second responses as failures and retry. Your 200 has to come back
  fast — process asynchronously if needed.
- **You can't ask the sender to fix their payload.** Schema drift, missing
  fields, surprise event types are things you have to tolerate, log, and
  triage on your side.

## Required sections (in addition to the common skeleton)

### 4a. Endpoint & contract

- Per sender (Stripe / GitHub / partner X): URL, HTTP method, expected
  content type, expected event types.
- How event types are dispatched (single endpoint with type-on-payload, or
  multiple endpoints).
- Acknowledged-by-200 vs other status (does the sender retry on non-2xx,
  on timeout, on 5xx only?).
- The exact response latency expectation of each sender (Stripe is ~10s
  for example — check the docs of each).

### 4b. Authenticity verification

- **Signature verification** — the canonical mechanism (HMAC over body +
  timestamp; library-supplied verification). Where the secret is stored,
  rotation story.
- **Replay protection** — timestamp window + nonce store. State the
  window (e.g. 5 minutes).
- **TLS** — sender expectations on cert validity, mTLS if available.
- **Backup auth** — IP allowlist if the sender publishes a stable range
  (defense in depth; not a substitute for signatures).

### 4c. Receive-and-acknowledge fast

The 200 must come back inside the sender's timeout. To achieve this:

- Verify signature (cheap).
- Persist the **raw payload** to durable storage immediately (write-ahead
  log: DB row or queue).
- Return 2xx.
- Process the payload asynchronously (queue + worker — see
  `references/tdd-async-worker.md`).

Avoid doing the actual business logic inline before the 200. If a
downstream is slow, you'll start failing webhooks and the sender will
re-deliver, amplifying load.

### 4d. Idempotency / dedup

- Idempotency key — sender-provided event ID is the canonical choice
  (Stripe's `id`, GitHub's `X-GitHub-Delivery`).
- Dedup store — a table or cache of recently-seen IDs with TTL matching
  the sender's retry window.
- What happens on a duplicate — return 200 (acknowledge), do nothing
  downstream.
- What if the sender doesn't supply an ID — synthesize from
  `hash(body + canonical_timestamp)`; document the risk of replay within
  the same second.

### 4e. Out-of-order & late events

- Are events causally ordered? Most senders make no global ordering
  guarantee.
- Per-entity ordering — process events for the same entity in arrival
  order? Or accept any order and reconcile by event timestamp?
- Late-arriving event handling — if a "cancelled" arrives after the
  "succeeded" for the same entity, what wins?

### 4f. Schema tolerance

- Unknown event types — log and 200, don't error (or sender starts
  retrying forever on something you don't handle).
- Unknown fields — ignore (forward-compatible).
- Missing-but-expected fields — fail closed and DLQ for triage.
- A schema-version field (if the sender publishes one) — handle
  per-version dispatch.

### 4g. Async processing (after the 200)

- Queue and worker design (see `tdd-async-worker.md`).
- Retry policy and DLQ — same shape as other workers.
- Downstream side-effects — idempotent.

### 4h. Observability

- Per-sender / per-event-type rate, success rate, DLQ rate.
- p95 receive-to-200 latency.
- Signature-failure rate (security signal — sudden spike = key rotation
  issue or attack).
- Sender-side retry rate (if visible).

## NFRs that matter for webhooks

From `nfrs-checklist.md`:

- **Availability** — high (sender retries are expensive; outages can
  cause sender backoff and ultimately webhook disablement). 99.9+ typical.
- **Latency** — receive-to-200 p95 must be inside the sender's timeout
  (often <5s, ideally <1s).
- **Security** — full threat model. Public internet, signed payloads,
  replay window, secret rotation.
- **Reliability** — durability of the raw payload before 200 returned;
  zero-loss target.
- **Observability** — per-sender breakdown is critical (each sender is
  its own incident surface).

## Common pitfalls

- **Doing business logic inline before the 200.** A slow DB makes the
  endpoint slow, the sender times out, retries, and load compounds.
- **No signature verification** — anyone on the internet can inject events.
- **No replay protection** — captured payload can be replayed indefinitely.
- **No dedup on event ID** — every sender retry causes double-effects.
- **Returning 5xx on unknown event type** — the sender retries that
  unknown type forever; floods your logs.
- **One endpoint, one secret across multiple sender environments**
  (sandbox / prod confusion).
- **Secret in code / env without rotation plan** — sender rotates, you
  don't, all webhooks start failing simultaneously.
- **Hosted on a serverless function with cold starts** — sender retries
  during the cold-start window, dedup may not catch the duplicate if both
  fire concurrently.

## Key decisions to surface in "Alternatives considered"

- Per-sender endpoint vs single shared endpoint with dispatch.
- Inline processing vs receive-and-enqueue (almost always: enqueue).
- Where the dedup store lives (DB unique constraint, Redis with TTL).
- Synchronous re-attempt vs hand off to async retry (rarely sync).
- Whether to expose your webhook URL directly or behind a CDN/WAF (often:
  behind, for signature-aware filtering and DDoS protection).
