# Phase 0.1 — Prerequisites checklist (for the human)

Nothing in Phase 0.2+ can run until these exist. Tick each one, then tell Claude Code "prerequisites done"
and paste the non-secret values (never paste API keys into chat or git — put them in the VPS `.env`).

## 1. Domain on Cloudflare (needed for the private URL)
- [ ] Own a domain (any cheap one; it's only for `studio.<domain>`).
- [ ] Add it to a free Cloudflare account and switch its nameservers to Cloudflare.
- [ ] Turn on Cloudflare Zero Trust (free plan) — it asks for a team name; note it.
- **Give back:** the domain name, the Zero Trust team name.

## 2. VPS (always-on home for the app + gallery)
- [ ] Rent a small Linux VPS: Ubuntu LTS, 1–2 vCPU, 2 GB RAM, 40+ GB disk (gallery grows over time).
- [ ] Add your SSH public key when creating it (no password login).
- **Give back:** the VPS IP and the login username. (Not the private key.)

## 3. RunPod (serverless GPU + training pod)
- [ ] Create a RunPod account and add a small credit (e.g. $10–20) to start.
- [ ] Create an API key → store it only in the VPS `.env` later (`RUNPOD_API_KEY`).
- [ ] Choose one datacenter with good availability of 24 GB+ GPUs; the network volume and all endpoints will be pinned to it.
- **Give back:** the chosen datacenter name. (Not the key.)

## 4. Container registry (where worker images are pushed by GitHub Actions)
- [ ] Use GitHub Container Registry (ghcr.io) — free with this repo; no extra account.
- [ ] Nothing to do now; Phase 1 sets the Actions secrets.

## 5. Consent evidence (required before Phase 6 training — start now)
- [ ] Her written consent (signed note, message, or form) covering: training a likeness model of her, generating images of her, personal use only.
- [ ] Save it as a file (PDF/image). It will be stored as the consent record's evidence, not in git.

## 6. Accounts you'll use on the phone
- [ ] The one email address that will be allowed through Cloudflare Access (only this email gets the login code).
- **Give back:** that email address.
