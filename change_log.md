# CHANGE LOG — append-only, newest first
# Format:
# ## <YYYY-MM-DD HH:MM> — [Phase X.Y] <title>
# - Role / Branch / Changed / Files / Artifact / Why / Verification(PASS|FAIL) / Follow-ups
---
## 2026-10-25 — [DECISION] [Phase 0] VPS replaced by Cloudflare serverless (plan.md v3.1)
- Role: manager
- Branch: phase-0-infra
- Changed: hosting moved from a VPS to Cloudflare Workers + D1 + R2 (free tier, pay-per-use); backend becomes a
  TypeScript Worker; job completion via RunPod webhook + reconcile cron; GPU worker watermarks and uploads to R2
- Files: /plan.md (D4, D9, D16, new D18; §1, §3, §4, §6, Phases 0–5/7/8, §8–§14), /.dev.vars.example (was
  /.env.example), /.gitignore, /infra/PREREQUISITES.md
- Artifact: plan.md v3.1; updated checklist
- Why: a VPS is billed for every hour it exists; human wants pay-per-use. Verified first: RunPod `/run` accepts a
  `webhook` field (2 retries, async results kept ~30 min); free Workers = 100k requests/day, 10 ms CPU, 5 cron triggers
- Verification: PASS — no VPS references left in plan.md outside the revision notes (grep)
- Follow-ups: human completes the revised checklist; then 0.2 Worker skeleton

## 2026-10-24 20:26 — [Phase 0.1] Prerequisites checklist written
- Role: infra-agent
- Branch: phase-0-infra
- Changed: listed everything the human must set up before VPS/Cloudflare/RunPod work
- Files: /infra/PREREQUISITES.md
- Artifact: infra/PREREQUISITES.md
- Why: Phase 0.2+ cannot run without a domain, VPS, RunPod account, and allowed email
- Verification: PASS — checklist present; covers domain, VPS, RunPod, registry, consent, allowed email
- Follow-ups: BLOCKED on human — complete checklist and return the non-secret values

## 2026-10-24 20:26 — [Phase 0.0] On-disk control and config files materialized
- Role: infra-agent
- Branch: phase-0-infra (off main d5276f2)
- Changed: wrote §9 and §11 files to the repo root
- Files: /.gitignore, /.env.example, /handoff.md, /change_log.md
- Artifact: the four files at repo root
- Why: lock repo hygiene and the living control files before building
- Verification: PASS — files exist and parse (see check in commit)
- Follow-ups: 0.1 prerequisites checklist

## 2026-10-24 20:20 — [Phase 0.0-pre] Repository initialized
- Role: manager
- Branch: main
- Changed: added project constitution and session instructions
- Files: /plan.md, /CLAUDE.md
- Artifact: commit d5276f2 on main
- Why: lock the operating protocol before building
- Verification: PASS — files present on origin/main
- Follow-ups: infra-agent → Phase 0.0
