Read the TDD at `/Users/nhoze/.claude/skills/tech-lead/evals/tdd-order-events-consumer.md`. Covers a Kafka-driven `order-projections` consumer, idempotent Postgres projection with `(partition, offset)` log, DLQ + replay/back-fill tooling, and an internal read API. 2 open questions noted (replay behavior under schema migrations; whether per-customer lookups need sub-100ms reads). Moving on to destination selection.

You asked me to create the tickets in Jira, but I don't see a Jira / Atlassian MCP available in this session. Three options:

1. **Set one up now (interactive).** I can walk you through installing a Jira/Atlassian MCP — I'll point you at the most current options (Anthropic's MCP directory, smithery.ai, the `modelcontextprotocol/servers` GitHub org), you handle the Atlassian site URL + email + API token, edit the MCP config (e.g. `~/.claude.json` on Claude Code), restart, run `/mcp` to verify, and we resume ticket creation once it's available.
2. **Skip MCP, give me paste-ready output.** I'll still produce the full breakdown; you'll get one structured block per task that you paste into Jira by hand, in dependency order.
3. **Switch to the local directory option** and I'll write the tasks to `./tasks-order-projections/` in the working dir (one markdown file per task plus a `README.md` index) instead.

Which would you like?
