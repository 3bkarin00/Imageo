# Phase 0.1 — Prerequisites checklist (for the human)

Nothing in Phase 0.2+ can run until these exist. Tick each one, then tell Claude Code "prerequisites done"
and paste the non-secret values (never paste API keys into chat or git — Claude Code will ask you to run
`wrangler secret put` for each secret yourself).

No VPS is needed (plan.md v3.1, D9): the app runs on Cloudflare's free tier, pay-per-use.

## 1. Cloudflare account + domain (needed for the private URL)
- [ ] Create a free Cloudflare account: https://dash.cloudflare.com/sign-up
- [ ] Buy a domain through Cloudflare Registrar (already on Cloudflare, sold at cost): https://domains.cloudflare.com
      — or add a domain you own elsewhere and switch its nameservers to Cloudflare.
- [ ] Turn on Cloudflare Zero Trust (free plan): https://one.dash.cloudflare.com — it asks for a team name; note it.
- [ ] Enable R2 (dashboard → R2). Cloudflare likely asks for a payment method (card or PayPal) even though usage
      stays inside the free tier for one user. If your card is refused, tell Claude Code before continuing.
- **Give back:** the domain name, the Zero Trust team name, and whether R2 is enabled.

## 2. RunPod (serverless GPU + training pod)
- [ ] Create a RunPod account: https://www.runpod.io and add a small credit (e.g. $10–20) in https://console.runpod.io
- [ ] Create an API key (guide: https://docs.runpod.io/get-started/api-keys) and keep it private.
- [ ] Choose one datacenter with good availability of 24 GB+ GPUs; the network volume and all endpoints will be pinned to it.
- **Give back:** the chosen datacenter name. (Not the key.)

## 3. GitHub (already have it)
- [ ] Repo: https://github.com/3bkarin00/Imageo — Claude Code will tell you which Actions secrets to add
      (a scoped Cloudflare API token, account ID) when Phase 0.5 needs them.
- [ ] Container registry: GitHub Container Registry (ghcr.io) — nothing to do now.

## 4. Consent evidence (required before Phase 6 training — start now)
- [ ] Her written consent (signed note, message, or form) covering: training a likeness model of her, generating
      images of her, personal use only.
- [ ] Save it as a file (PDF/image). It becomes the consent record's evidence, stored privately — not in git.

## 5. The one allowed email
- [ ] The email address that will be allowed through Cloudflare Access (only this email gets the login code).
- **Give back:** that email address.
