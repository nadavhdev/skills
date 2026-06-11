# TASK-04: Email provider integration

## Outcome
Service can send a templated email to a user via the chosen provider (SES / SendGrid / etc.) behind a clean internal interface.

## Scope
- Decide provider (open question). Default assumption: SES.
- Implement `EmailSender` interface with `send(to, template_key, variables, locale) -> SendResult`.
- Template rendering: subject + HTML body + plain-text fallback. Use `notification_template` table from TASK-02.
- DKIM/SPF/DMARC verification for sending domain (coordinate with infra).
- Provider webhook receiver for bounces and complaints: `POST /v1/email-events` (signature-verified) — marks user email as suppressed.
- Suppression list table or column on user record (coordinate with TASK-07).

## Acceptance criteria
- [ ] Sandbox send produces a delivered email in a test inbox with correct rendering on Gmail + Outlook + Apple Mail.
- [ ] Bounce webhook test produces a suppression entry; suppressed addresses are not retried.
- [ ] Plain-text alt is generated for every HTML template.
- [ ] DKIM/SPF/DMARC pass on a real send (verify via mail-tester or similar).

## Dependencies
- TASK-01, TASK-02. Blocks: TASK-06.

## Out of scope
- Promo campaign API (TASK-06).
- Per-user channel preferences (TASK-07).

## Estimate
~4-5 days
