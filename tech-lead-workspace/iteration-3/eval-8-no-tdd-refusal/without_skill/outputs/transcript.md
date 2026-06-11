Done — I broke the notifications service into 10 ticket-sized tasks and wrote them to `./tasks-notifications-service/` in the working directory. Each is a standalone markdown file with outcome, scope, acceptance criteria, dependencies, and a rough estimate, so the team can pull them straight into your tracker.

## Where things live

```
tasks-notifications-service/
  README.md                          <- sequencing diagram + sprint plan + open questions
  TASK-01-service-skeleton.md
  TASK-02-data-model.md
  TASK-03-push-provider.md
  TASK-04-email-provider.md
  TASK-05-order-shipped-push.md      <- the "order shipped push" use case
  TASK-06-promo-campaign-email.md    <- the "promo campaign email" use case
  TASK-07-user-preferences.md
  TASK-08-delivery-retries-dlq.md
  TASK-09-observability.md
  TASK-10-rollout.md
```

## How I sliced it

The two product use cases — order-shipped push and promo campaign email — each need a stack underneath them, so I split foundation from feature:

- **Foundation (Sprint 1):** service skeleton, data model, user preferences. Without these, anything else is throwaway.
- **Channels (Sprint 2):** push provider + email provider as separate tickets so they can run in parallel and so we can pick providers independently. Retries/DLQ rides alongside.
- **Use cases (Sprint 3):** the actual order-shipped consumer (TASK-05) and the promo campaign API + bulk sender (TASK-06). These are the user-visible deliverables.
- **Production-ready (Sprint 4):** observability and a staged rollout with feature flags + runbook. Non-negotiable before turning either channel on for real users.

## Sprint plan (rough)

- **Sprint 1:** TASK-01, TASK-02, TASK-07
- **Sprint 2:** TASK-03, TASK-04, TASK-08
- **Sprint 3:** TASK-05, TASK-06
- **Sprint 4:** TASK-09, TASK-10

## Open questions I called out in the README

These should be answered in or before Sprint 1 — they don't block starting, but they do block finishing:

1. **Push provider** — FCM/APNs direct, or a vendor (OneSignal, Braze)?
2. **Email provider** — SES, SendGrid, Postmark, Mailgun?
3. **Event bus** — what does orders publish on today? (Kafka, SNS+SQS, RabbitMQ?)
4. **Promo API auth model** — internal service-to-service only, or wired to an admin UI?
5. **Compliance** — do we already have a central consent/unsubscribe store, or are we building it here (TASK-07)?

## A few things I deliberately left out of scope

So you can decide whether to add tickets:

- Transactional emails other than order-shipped (receipts, password resets) — these will eventually want to ride this same service but aren't in the ask.
- In-app/SMS channels.
- A/B testing, scheduled recurring campaigns, rich segmentation UI.
- Customer-facing campaign analytics (open rate, CTR).
- Quiet hours / frequency caps.

If any of those are actually in scope, let me know and I'll add tickets. Otherwise the team should be unblocked to start grooming Sprint 1.
