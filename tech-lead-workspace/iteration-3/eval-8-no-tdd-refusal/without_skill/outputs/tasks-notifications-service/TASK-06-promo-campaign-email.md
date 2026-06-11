# TASK-06: Promo campaign API + bulk email sender

## Outcome
Marketing (or an admin tool) can launch a promo email campaign to a defined audience, and the system delivers it in a paced, observable, compliant way.

## Scope
- API:
  - `POST /v1/campaigns` — create campaign (name, template_key, audience filter, schedule_at).
  - `POST /v1/campaigns/{id}/launch` — kick off sending.
  - `GET /v1/campaigns/{id}` — status + counts (queued, sent, bounced, suppressed).
- Audience resolution: query users matching filter (start simple — `all_users`, `country=X`, `segment=Y`); paginate.
- Fan-out: enqueue one job per recipient onto a queue; worker calls `EmailSender`.
- Rate limiting: respect provider send-rate quota (token bucket per campaign + global).
- Compliance:
  - Honor suppression list and unsubscribe (TASK-07).
  - Include a working unsubscribe link in every promo email.
  - Exclude transactional-only users (per opt-in flag).

## Acceptance criteria
- [ ] Launching a 10k-user test campaign in staging completes within agreed window without exceeding provider rate limit.
- [ ] Unsubscribed and suppressed users are excluded; counts reported on campaign status.
- [ ] Failed sends are retried per TASK-08 policy.
- [ ] Campaign cannot be launched twice (idempotent launch).

## Dependencies
- TASK-02, TASK-04, TASK-07, TASK-08.

## Out of scope
- A/B testing, scheduled recurring campaigns, rich segmentation UI.
- Transactional emails (order receipts, password resets) — separate epic.

## Estimate
~6-8 days
