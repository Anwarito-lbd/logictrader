# RUNBOOK

## Start
1. `mise install` (if `mise.toml` exists)
2. `task setup`
3. `task validate`

## Daily Operator Loop
1. Pick highest-impact task tied to `MONEY_KPI.md`
2. Implement smallest shippable change
3. Run `task validate`
4. Log outcome in PR/commit notes

## Deploy
- Use repo-specific deploy command (document in README).
- Verify health checks after deploy.

## Rollback
- Revert last release/commit.
- Re-run `task validate`.
- Restore previous stable deployment.
