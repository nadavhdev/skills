I won't break this down without a TDD — without one, the tasks would be my opinions about a feature, not a decomposition of a design the team has agreed on, and the result wouldn't be principal-level. What you've given me here ("push notifications when orders ship, emails for promo campaigns") is a feature blurb, not a design.

To move forward, please do one of the following:

1. **Paste the TDD** for the notifications service here, or
2. **Give me a file path or URL** I can fetch (local file, public URL, or a Drive/Confluence link with the relevant MCP configured), or
3. **If you only have a PRD** (or no design doc at all yet), say so and I'll author the TDD first via the TDD-authoring capability — then we break that down into tickets.

A real TDD would need to cover, at minimum: the design for how shipment events reach the push channel (event source, delivery guarantees, idempotency, provider — APNs/FCM/etc.), how promo campaigns are authored, scheduled, segmented, and sent (and how unsubscribes/suppression are honored), the data model for devices/tokens and user contact preferences, retry/failure semantics, rate limits against the upstream providers, and the NFRs (throughput for campaigns, latency for transactional shipment pushes, deliverability observability).

Without that design committed, any tickets I write would be guesses dressed up as a sprint plan. Send the TDD (or the PRD, and I'll design it first) and I'll break it down.
