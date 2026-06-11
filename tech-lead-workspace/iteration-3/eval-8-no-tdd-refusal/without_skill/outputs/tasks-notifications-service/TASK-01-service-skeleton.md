# TASK-01: Stand up notifications service skeleton

## Outcome
A deployable `notifications-service` exists in our infra with health checks, CI, and a "hello world" HTTP endpoint, so subsequent tickets have a home for their code.

## Scope
- New repo (or new module in monorepo, per team convention).
- HTTP server with `/healthz` and `/readyz`.
- Dockerfile + base CI pipeline (lint, test, build, image push).
- Deploy to dev environment via existing IaC pattern.
- Config loader for env vars (12-factor): provider keys, queue URLs, DB URL placeholders.

## Acceptance criteria
- [ ] Service builds and deploys to dev via CI on merge to main.
- [ ] `/healthz` returns 200; `/readyz` returns 200 once dependencies (none yet) are reachable.
- [ ] README documents local run, env vars, deploy command.
- [ ] Logs are structured (JSON) and shipped to existing log aggregator.

## Dependencies
- None. Blocks: TASK-02 through TASK-10.

## Out of scope
- Any actual notification logic.
- Persistence layer (TASK-02).

## Estimate
~3 days
