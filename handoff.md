# HANDOFF — read FIRST, rewrite LAST
## Current state
- Phase: 0 — Infrastructure & access
- Step: 0.2 — VPS hardening script (`infra/setup_vps.sh`)
- Status: BLOCKED (waiting on human prerequisites)
- Owner role: infra-agent
- Branch: phase-0-infra
- Last updated: 2026-10-24 20:26 by infra-agent
## Done so far
- 0.0 PASS — .gitignore, .env.example, handoff.md, change_log.md
- 0.1 PASS — infra/PREREQUISITES.md
## Next action (do exactly this)
When the human confirms prerequisites, write `infra/setup_vps.sh` (updates, firewall SSH-only, key-only SSH,
no password login, non-root user, Python, systemd unit stubs), verifying every command against the VPS's
installed Ubuntu version before relying on it.
## Acceptance for this step
Script runs cleanly on the VPS; `ss -tlnp` / firewall status show only SSH listening publicly (saved output).
## Blockers
- Human must complete infra/PREREQUISITES.md and provide: domain name, Cloudflare Zero Trust team name,
  VPS IP + username, chosen RunPod datacenter, the one allowed email. Secrets go only in the VPS `.env`.
## Notes
- No phase starts before its predecessor has a PASS in change_log.md.
- Every acceptance check produces a named artifact; no mocks, no assumed API signatures.
- After each step: commit + push to the phase branch; merge to `main` only on PASS (§13).
