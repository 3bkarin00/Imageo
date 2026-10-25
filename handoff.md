# HANDOFF — read FIRST, rewrite LAST
## Current state
- Phase: 0 — Infrastructure & access
- Step: 0.2 — Worker skeleton + `wrangler.toml`
- Status: BLOCKED (waiting on human prerequisites)
- Owner role: infra-agent
- Branch: phase-0-infra
- Last updated: 2026-10-25 by manager
## Done so far
- 0.0 PASS — .gitignore, env template (now .dev.vars.example), handoff.md, change_log.md
- 0.1 PASS — infra/PREREQUISITES.md (revised for v3.1: no VPS)
- [DECISION] plan.md v3.1 — hosting is Cloudflare Workers + D1 + R2 (see change_log)
## Next action (do exactly this)
When the human confirms prerequisites: create the Worker skeleton in `server/` (TypeScript + Hono) and
`wrangler.toml` with D1 + R2 bindings, custom domain `studio.<domain>`, `workers_dev = false`, preview URLs off;
deploy a "hello" page. Verify every wrangler setting and command against current Cloudflare docs and the
installed wrangler version before relying on it.
## Acceptance for this step
`https://studio.<domain>` serves the hello page; the `*.workers.dev` address does not (saved curl output).
## Blockers
- Human must complete infra/PREREQUISITES.md and provide: domain name, Zero Trust team name, R2 enabled (yes/no),
  chosen RunPod datacenter, the one allowed email. Secrets are set by the human with `wrangler secret put`, never pasted.
## Notes
- No phase starts before its predecessor has a PASS in change_log.md.
- Every acceptance check produces a named artifact; no mocks, no assumed API signatures.
- After each step: commit + push to the phase branch; merge to `main` only on PASS (§13).
