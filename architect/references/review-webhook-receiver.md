# Architect review aspects — webhook receiver (inbound)

## Rubric to review against
Read `~/.claude/skills/tech-lead/references/tdd-webhook-receiver.md` first — that
is the standard this TDD was (or should have been) written against. Use its
required sections, NFR applicability, and pitfalls as the baseline checklist.
This file adds the architect lens.

## The one-way doors (scrutinize hardest — expensive to reverse)
- **The public endpoint contract.** The URL, the accepted payload shape, the
  response semantics — a third-party provider is configured against these.
  Changing them means coordinating with an external party.
- **Signature-verification scheme.** How you authenticate that the call really
  came from the provider. Foundational security; retrofitting is painful.
- **Sync-ack vs async-process boundary.** Whether you do work in the request or
  ack-fast-then-process. Reshaping this changes the whole flow.

## Type-specific hot spots (review questions)
- **Signature / HMAC verification at the trust boundary.** This is an untrusted
  inbound request from the public internet. No verification → `security`
  `blocker`. Is the secret stored and rotatable?
- **Idempotency.** Providers retry and send duplicates. Is there a dedup key
  (event id) and defined behavior, or will duplicates double-process
  (`data`/`risk`)?
- **Fast-ack, then process async.** Heavy work inside the request causes provider
  timeouts and retry storms. Does the design ack quickly and hand off?
- **Replay-attack protection.** Timestamp window / nonce so a captured request
  can't be re-sent later (`security`).
- **Out-of-order delivery & payload validation.** Webhooks arrive out of order;
  payloads are attacker-influenced. Both validated?

## Over-engineering & simplification tells
- A broker / streaming platform stood up for low-volume webhooks where a simple
  async handler + table would do (`over-engineering`).
- A bespoke retry/queue layer where the codebase's existing async-worker
  infrastructure fits (`codebase-fit`).

## Calibration — NFRs
- **Demand if absent:** signature verification, idempotency, fast-ack/async
  split, replay protection, payload validation, an **availability SLO** (senders
  back off and ultimately disable webhooks on repeated failure), and a **test
  strategy** covering duplicate, replay, and unknown-event-type cases. Security
  gaps here are `blocker`s by the hard severity rule — this is a public trust
  boundary.
- **Flag as over-reach if present:** heavy streaming infra or multi-region for a
  modest webhook volume.
