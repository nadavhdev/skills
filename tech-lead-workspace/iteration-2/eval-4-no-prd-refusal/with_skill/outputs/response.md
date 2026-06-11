I won't draft a TDD without a PRD — designing without one would mean me inventing the requirements rather than designing for them, and the result wouldn't be principal-level. Please paste the PRD here, give me a file path or URL I can fetch, or let me know if you'd like help writing the PRD first.

A few notes on why what you've shared isn't enough to design against, so you know what to bring tomorrow:

- "Push when an order ships" and "emails for promo campaigns" are two materially different workloads (transactional, event-driven, per-user vs. bulk, scheduled, campaign-shaped). They have different delivery semantics, different idempotency stories, different failure modes, and different vendor choices. A real PRD would scope which one(s) are in this release and the priority between them.
- "100k users growing to 1M" is a user count, not a load profile. I'd need send rates (notifications/sec at peak for shipping events, emails/hour for a campaign blast), retention of delivery history, and the latency budget from "order shipped" to "device buzzes".
- The PRD also needs to state: who owns user contact info and consent/unsubscribe state, which channels are required per notification type, what the fallback is when push fails (SMS? email? nothing?), and the legal surface (CAN-SPAM, GDPR, per-region consent).

If you can paste the PRD, drop a file path, or share a Drive/Confluence link (tell me which MCP you have configured for it), I can turn it around well before tomorrow's review. If there isn't a PRD yet, I'm happy to help you write one first — that's usually faster than redoing the TDD after the review surfaces the gaps.
