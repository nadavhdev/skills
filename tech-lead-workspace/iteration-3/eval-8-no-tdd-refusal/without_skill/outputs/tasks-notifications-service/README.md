# Notifications Service — Task Breakdown

Feature scope: users receive **push notifications** when their order ships, and **emails** for promotional campaigns.

This directory contains the ticket-sized breakdown for the team to start sprinting. Each `TASK-XX.md` is a self-contained ticket with outcome, scope, acceptance criteria, and dependencies. No implementation code — that's for the engineer who picks up the ticket.

## Suggested sequencing

```
TASK-01 (infra/service skeleton)
   |
   +--> TASK-02 (data model + persistence)
   |        |
   |        +--> TASK-03 (push provider integration)
   |        +--> TASK-04 (email provider integration)
   |                  |
   |                  +--> TASK-05 (order-shipped event consumer -> push)
   |                  +--> TASK-06 (promo campaign API + sender -> email)
   |
   +--> TASK-07 (user preferences + opt-out)
   +--> TASK-08 (delivery tracking + retries/DLQ)
   +--> TASK-09 (observability: metrics, logs, alerts)
   +--> TASK-10 (rollout: feature flag, staged release, runbook)
```

## Sprint guidance

- **Sprint 1 (foundation):** TASK-01, TASK-02, TASK-07
- **Sprint 2 (channels):** TASK-03, TASK-04, TASK-08
- **Sprint 3 (use cases):** TASK-05, TASK-06
- **Sprint 4 (production-ready):** TASK-09, TASK-10

## Open questions to resolve before/during Sprint 1

- Which push provider? (FCM / APNs direct, or a vendor like OneSignal / Braze)
- Which email provider? (SES, SendGrid, Postmark, Mailgun)
- Is there an existing event bus / queue (Kafka, SNS+SQS, RabbitMQ) we should publish/consume on?
- Auth model for the promo campaign API — internal-only (marketing tool) or also exposed via admin UI?
- Compliance: CAN-SPAM / GDPR / CASL — do we already have a central unsubscribe/consent store?
