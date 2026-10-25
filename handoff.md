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
- [DECISION] plan.md v3.1 — hosting is Cloudflare Workers + D1 + R2; v3.2 — watermark removed (see change_log)
## Next action (do exactly this)
When the human confirms prerequisites: create the Worker skeleton in `server/` (TypeScript + Hono) and
`wrangler.toml` with D1 + R2 bindings, custom domain `studio.<domain>`, `workers_dev = false`, preview URLs off;
deploy a "hello" page. Verify every wrangler setting and command against current Cloudflare docs and the
installed wrangler version before relying on it.
## Acceptance for this step
`https://studio.<domain>` serves the hello page; the `*.workers.dev` address does not (saved curl output).
## Known values (from the human, 2026-10-25)
- Domain: imageoo.party → app at `https://studio.imageoo.party`
- Cloudflare Zero Trust: enabled. R2: enabled.
- RunPod: account funded; network volume already created, id `d61ym2xhuy` (record its datacenter when first used).
- RunPod API key: the first key was exposed in a chat and must be revoked; human creates a new one and sets it
  only via `wrangler secret put RUNPOD_API_KEY` in their own terminal. Never ask for it in chat.
## Blockers
- Still needed from the human: Zero Trust team name, the network volume's datacenter, the one allowed email.
## Notes
- No phase starts before its predecessor has a PASS in change_log.md.
- Every acceptance check produces a named artifact; no mocks, no assumed API signatures.
- After each step: commit + push to the phase branch; merge to `main` only on PASS (§13).
