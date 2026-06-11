Happy to write this TDD, but before I draft I want to surface a few things so we don't end up with a doc that gets sent back at the review.

**On the PRD.** What you've given me is a one-paragraph feature brief, not a PRD. That's fine for an opening pitch, but a few of the answers materially change the architecture (transactional push vs. campaign email are very different beasts at 1M users), so drafting without them means I'd be inventing requirements. If there's an actual PRD doc — Notion, Google Doc, Confluence, a markdown file — point me at it and I'll pull from it. If there isn't one yet, that's also fine; I just want to flag that this TDD will be built on the answers you give me below, not on product sign-off.

**On project conventions.** I checked the current working directory and the only `CLAUDE.md` I can find is for `csvkit` (a CSV CLI suite), which is unrelated to a notifications service — so there are no project conventions I can honor here. If this notifications service lives in a different repo with its own `CLAUDE.md` / `ARCHITECTURE.md` / `CONTRIBUTING.md`, point me at that repo and I'll re-read before drafting; otherwise I'll treat this as greenfield and omit the "Design constraints from project conventions" section of the TDD.

**On the workload classification.** Your brief actually describes **two different workloads** that I'd normally write as two TDDs:

1. **Transactional push notifications** on "order shipped" — this is an **event-driven consumer** (something publishes an `order.shipped` event, the notifications service consumes it and fans out to APNs/FCM). Low-latency, per-user, must-not-lose, idempotency matters.
2. **Promotional email campaigns** — this is a **batch / async-worker** pattern (marketing schedules a campaign for a segment of N users, workers send N emails through SendGrid/SES with rate-limit backpressure). High-volume bursts, eventual delivery, unsubscribe + compliance dominate.

They share an outbound delivery abstraction but very little else (different SLAs, different failure modes, different cost shapes, different on-call concerns). My default would be **one TDD covering both with a shared "delivery core" section + two workload-specific sections**, unless you tell me otherwise. Let me know if you'd rather have two separate docs.

**Triage questions I need answered before drafting** (keeping this to the ones that flip the design — everything else goes in the TDD's Open Questions section):

1. **Event source for "order shipped".** Is there an existing event bus (Kafka, SNS+SQS, PubSub, EventBridge, RabbitMQ) the order service already publishes to? Or am I designing the integration with the order service too? This decides whether I'm specifying a consumer or a contract.
2. **Push delivery provider.** Are we going direct to APNs/FCM, or through a managed provider (Firebase Cloud Messaging end-to-end, OneSignal, Braze, Customer.io, AWS Pinpoint)? Buy vs. build is a huge fork — managed providers collapse most of the design.
3. **Email delivery provider.** Same question — SES, SendGrid, Postmark, Mailgun, or a marketing platform like Braze/Iterable/Customer.io? For 1M-user campaigns, the provider's send-rate limit and warmup behavior become the dominant constraint.
4. **Campaign authoring + scheduling.** Who creates a campaign and how — marketing logs into a UI we build, calls an API, drops a CSV, or it comes from an existing marketing tool? This decides whether we're building a CMS-lite or just an executor.
5. **Cloud + language constraints.** What cloud (AWS / GCP / Azure / on-prem)? What's the team's language stack (so I'm not designing a Go service for a Python team)? Any K8s / serverless preference?
6. **Compliance + consent.** GDPR (EU users)? CAN-SPAM / CASL for promo email (unsubscribe required, double opt-in)? TCPA if you ever extend to SMS? Where does the consent/preferences store live, and is it in scope for this service or upstream?

A few things I'm **not** asking upfront because I can put reasonable defaults in the TDD and flag them as Open Questions:

- Exact latency budget for transactional push (I'll assume "within a few seconds of `order.shipped`" — fine for shipping notifications, not fine for 2FA).
- Retry / DLQ policy specifics.
- Per-user preference granularity (per-channel opt-out vs. per-category).
- Multi-region / DR posture (at 100k–1M users with non-critical notifications I'd default to single-region with cross-AZ).
- Cost ceiling — I'll size it but you'll need to confirm.

**On the timeline.** "I need it for tomorrow's review" is doable, but a TDD whose six core questions are still open will get torn apart in review. If you can answer the six above in a single reply tonight, I'll have a solid draft ready well before your review. If you can only answer some, I'll draft against your answers and explicitly call out the unanswered ones as blockers in the doc rather than inventing answers.

Send the answers (or the PRD if one exists) and I'll write `TDD-notifications-service.md` in the current directory.
