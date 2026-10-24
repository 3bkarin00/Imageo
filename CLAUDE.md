# CLAUDE.md — imageo

This repo is built phase by phase under the constitution in `plan.md`. Every session:

1. Read `handoff.md` first (if it doesn't exist yet, you are at Phase 0, step 0.0 — create it from `plan.md` §11).
2. Read `plan.md` — follow §2 (protocol) and §4 (rules) exactly; settled decisions are in §5.
3. Do only the single next action in `handoff.md`, on the branch it names.
4. After each step: acceptance artifact → append `change_log.md` → rewrite `handoff.md` → commit + push on the phase branch.
5. Merge a phase branch into `main` only via PR, and only after its gate (§10) has a PASS in `change_log.md`.

Hard stops:
- Never assume an external API/CLI signature; verify against live docs, the running service, or the installed package.
- Never commit secrets, `.env`, model weights, datasets, or DB files.
- If a step needs the human (accounts, payment, domain, phone screenshots), write the exact request under Blockers in `handoff.md`, commit + push, and stop.
