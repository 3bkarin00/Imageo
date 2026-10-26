# HANDOFF — read FIRST, rewrite LAST
## Current state
- Phase: 0 — Infrastructure & access
- Step: 0.1 — Prerequisites checklist for the human (0.0 done: PASS)
- Status: BLOCKED on human
- Owner role: infra-agent
- Branch: claude/documentation-handoff-d7n3sx (plan says phase-0-infra; see Notes)
- Last updated: 2026-10-26 07:55 by infra-agent
## Next action (do exactly this)
When the human has supplied the Blockers below, write `infra/setup_vps.sh` (step 0.2): apt updates,
firewall (SSH only, key-only, no password login), non-root user, Python, systemd units. Verify each
command against the live VPS / installed packages — do not assume. Nothing else until 0.2 is PASS.
## Acceptance for this step
0.0 satisfied (files exist and parse). 0.1 is satisfied when the human confirms the items below.
## Blockers
Human must provide (reply in chat or commit to a private place — NEVER put secrets in git):
1. A domain whose DNS is managed in Cloudflare (free plan is fine) — tell me the domain; studio will live at `studio.<domain>`.
2. A Cloudflare account with Zero Trust enabled (free tier), and the single email to allow via Access one-time code.
3. A VPS (Ubuntu LTS, ~$5/mo, 1 vCPU/1 GB+) with: its IP, a non-root sudo user, and your SSH public key installed. Say which provider.
4. A RunPod account with a funded balance and an API key (store it only in the VPS `.env`, not in chat or git).
5. A container registry for worker images (GitHub Container Registry on this repo is the default; confirm).
6. Preferred RunPod datacenter for the network volume (pick for GPU availability; endpoints get pinned to it, D10).
## Notes
- 0.0 deviation: plan/handoff seed names branch `phase-0-infra`, but this session is restricted to pushing `claude/documentation-handoff-d7n3sx`. Work stays here until the human says to rename/branch.
- No phase starts before its predecessor has a PASS in change_log.md.
- Every acceptance check produces a named artifact; no mocks, no assumed API signatures.
- After each step: commit + push to the phase branch; merge to `main` only on PASS (§13).
