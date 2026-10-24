# plan.md — Personal AI Image Studio "imageo" (mobile PWA + cloud GPU) · v3

**This is the project constitution for building `imageo` with Claude Code** (repo:
`github.com/salahsterr/imageo`). It is self-contained: operating protocol, rules, decisions, agent
roster, phased plan with acceptance checks, interface contracts, Git workflow, and seeds for the living
files. Step 0.0 materializes the on-disk files. `CLAUDE.md` at the repo root points every Claude Code
session here.

> **Two files live OUTSIDE this one** (they change every step): `handoff.md` (where we are + next
> action) and `change_log.md` (append-only history). Seeds are in §11.

---

## 0. Revision log (v3 — informational; may be deleted once reviewed)
v3 is a **re-scope**, not an edit: the target changed from a local desktop studio to a personal
phone app backed by cloud GPUs. What changed vs v2.1:

- **Architecture inverted.** Local ComfyUI on your laptop → **PWA on your phone → small VPS (app,
  database, gallery) behind Cloudflare Access → RunPod Serverless GPU workers running ComfyUI.**
  Your laptop is now a dev machine only (§1, §5/D1–D4, D9–D10).
- **Builder changed:** Antigravity → **Claude Code**, working in the GitHub repo above.
- **Three-tab product defined:** Prompt (photo → JSON → neutralized prompt), Generate, Gallery (§1, Phase 7).
- **New: photo→JSON prompt service** with explicit-term **replacement** (not removal/blocking), on an open
  vision model running on your own GPU worker (Phase 5, D14–D15).
- **Training simplified** to a one-off scripted SDXL LoRA run on a RunPod GPU pod for one consented
  subject (Phase 6, D12). Training wizard and dataset health score deferred.
- **Base model = SDXL only.** FLUX.2 / Z-Image / InstantID removed from v1 (D2).
- **Deferred to later:** editor ops (inpaint/outpaint/instruction), pose composer, dataset health score,
  in-app training wizard, C2PA (§14).
- **Kept:** handoff/change_log protocol, artifact-backed acceptance, consent record, Git branch-per-phase.
- **New cost guard** (spend caps, max workers) because serverless billing is open-ended (§4, Phase 8).
- `manifest.yaml` from v2.1 is **out of scope for v3** and is rewritten in Phase 1 (§9).

---

## 1. North Star
A **private, single-user** image studio you use **from your phone**, anywhere:

1. **Prompt tab** — take/upload a photo → a vision model describes it as **structured JSON** → any
   explicit terms are **replaced with neutral equivalents** (shown to you) → one tap sends it to Generate.
2. **Generate tab** — text→image with your subject's LoRA, plus: **pose from a reference photo**,
   **image-to-image**, **face fix**, **upscale**.
3. **Gallery tab** — every generation with its prompt/JSON/settings; reuse, favorite, download, delete.

**We do not build a diffusion engine — we orchestrate ComfyUI**, running on RunPod Serverless.
Codenames: **`studio`** = the app we build (PWA + VPS backend). **`worker`** = the RunPod GPU side.

**System shape**
```
Phone (PWA) ──HTTPS──▶ Cloudflare Access (email one-time code, only you)
                         │  Cloudflare Tunnel (no open inbound web ports on the VPS)
                         ▼
                 VPS (~$5/mo): FastAPI + SQLite + gallery files + serves the PWA
                         │  RunPod API (key lives ONLY here)
                         ▼
        RunPod Serverless endpoints ── network volume (models, LoRA)
          • gen    : ComfyUI worker (SDXL, ControlNet, FaceDetailer, upscaler)
          • vision : open VLM → JSON description
        RunPod GPU pod (one-off) : LoRA training (kohya sd-scripts) → LoRA to network volume
```

---

## 2. Operating protocol (read every session, in this order)
1. **handoff.md** — where we are + the single next action. READ FIRST, REWRITE LAST.
2. **plan.md** (this file) — north star, rules, decisions, roster, phases, contracts.
3. **Agent roster (§6)** — the manager role dispatches the agent role that owns the current phase.
4. **RULES (§4)** — hard constraints.

**After EVERY step, in this exact order:**
a. run the step's acceptance check — it MUST produce the **named artifact** the check names (an image
   file, a JSON file, a DB row, a passing test, a screenshot from the phone). A check satisfied by a
   mock, a stub, or a claim without an artifact is a FAIL;
b. append one entry to `change_log.md` (newest first, fixed format), recording the artifact path;
c. rewrite `handoff.md` (new phase / step / status / next action);
d. commit + push on the **current phase branch**: `git add -A && git commit -m "[Phase X.Y] <title> — <PASS|FAIL>" && git push`.
   The commit message mirrors the change_log header; never commit gitignored weights/secrets/runtime (§9);
e. only then pick up the next step.

**Never** start a phase whose predecessor lacks a PASS entry in `change_log.md`.
**Never** assume an external API signature (RunPod endpoints/payloads, ComfyUI node inputs, kohya
flags, Cloudflare config, VLM output format) — verify it against the live service, docs, or installed
package first. A fabricated signature that "passes" a gate is the worst failure mode in this build (D13).
**Steps that need the human** (accounts, payments, domain, phone screenshots): stop, list exactly what
is needed in `handoff.md` under Blockers, commit + push, and wait.

---

## 3. File map
| Path | Role | Lifecycle |
|---|---|---|
| `CLAUDE.md` | Tells every Claude Code session to follow §2 | Static |
| `plan.md` | This constitution | Static; edit only via a decision |
| `handoff.md` | Current phase/step/next action | **Living** — rewritten each step |
| `change_log.md` | Append-only build history | **Living** — appended each step |
| `.gitignore`, `.env.example` | Repo hygiene, declared env vars (§9) | Static |
| `manifest.yaml` | Pinned worker deps, custom nodes, models (+license flags) | Rewritten Phase 1; bump via decision |
| `worker/gen/` | Dockerfile + handler for the ComfyUI serverless worker | Built Phase 1 |
| `worker/vision/` | Dockerfile + handler for the VLM serverless worker | Built Phase 5 |
| `training/` | Dataset prep + kohya config + run script for the one-off pod | Built Phase 6 |
| `server/` | FastAPI backend (VPS): api, runpod client, queue, data, prompt service | Phases 2–5 |
| `server/workflows/templates/` | ComfyUI API-format workflow templates | Phase 3 |
| `server/filters/neutral_map.yaml` | Explicit term → neutral term mapping | Phase 5 |
| `migrations/` | Versioned DB migrations | Grows with schema |
| `web/` | PWA (React + Vite), 3 tabs | Phase 7 |
| `infra/` | VPS setup script, cloudflared config, systemd units, backup script | Phases 0, 8 |
| `.github/workflows/` | CI: tests + build/push worker images | Phase 1 |
| `tests/` | Runnable gate checks (§10) | All phases |

---

## 4. RULES (hard constraints — violating these is a failed task)
**Architecture**
- studio NEVER implements diffusion/sampling/training loops. Generation = ComfyUI on the worker;
  training = pinned kohya sd-scripts on a pod; JSON description = an open VLM on the vision worker.
- **The VPS never runs models.** It orchestrates, stores, and serves. No torch on the VPS.
- All RunPod calls go through `server/runpod_client.py`. No raw RunPod HTTP elsewhere.
- All generation graphs are built by `server/workflows/builder.py` from template JSON + params,
  validated against an `/object_info` snapshot taken from the deployed worker image.
- `manifest.yaml` is the single source of truth for worker deps/nodes/models. Nothing installed or
  downloaded outside the worker Dockerfiles / volume-population script.

**Security (single user)**
- The PWA and API are reachable **only through Cloudflare Access**, allowing exactly one email (yours).
- The VPS accepts **no inbound traffic except SSH (key-only)**; web traffic arrives via Cloudflare
  Tunnel (outbound). The API process binds to `127.0.0.1`.
- **Secrets (RunPod API key, etc.) live only in the VPS `.env`** and in GitHub Actions secrets where CI
  needs them. Never in the PWA, never in git.
- The API verifies the Cloudflare Access JWT on every request (defense if the tunnel is misrouted).

**Cost guard (serverless is billed per use)**
- Serverless endpoints: **min/active workers = 0**, **max workers = 1**, short idle timeout.
- The backend enforces a **daily job cap** and a **monthly spend ceiling** (configurable); over the cap,
  jobs are refused with a clear message.
- The training pod is started by script and **auto-terminated** when training ends or fails.

**Operational**
- Jobs are queued **persistently** (SQLite); a VPS restart never loses a job. Every job can be cancelled.
- Every job is logged (structured, `var/logs/`): job id, intent, params, RunPod request id, status,
  error, cost estimate.
- Results are copied from the worker to VPS storage immediately; the gallery never depends on RunPod
  retaining outputs.

**Consent / trust**
- The subject is a real person: a **consent record** (her written consent, date, scope) must exist
  before training and before any generation using her LoRA (D8).
- Training photos are deleted from the network volume and pod after the LoRA is produced; the VPS keeps
  the originals only if you choose to.
- Every output carries an **invisible watermark** + its generation metadata in the DB.
- The explicit-term neutralizer runs on **every prompt sent to generation**, including text typed or
  edited in the Generate tab — not only on photo-derived JSON (D15).

**Bookkeeping**
- The LAST action of every task: append `change_log.md` (with artifact path) → rewrite `handoff.md` →
  commit + push. A task that skips this is not complete.

**Version control** — branch per phase; commit + push every step; merge to `main` only on PASS (§13).

---

## 5. Decisions (ADR-lite — settled; do not re-litigate)
- **D1 — Orchestrate ComfyUI, don't rebuild it** — on RunPod Serverless via its job API (`/run`, `/status`, `/cancel`).
- **D2 — Base model = SDXL only for v1.** Cheapest, fastest cold starts, deepest LoRA/ControlNet/FaceDetailer
  ecosystem. Use is personal-only, so non-commercial models would be allowed, but none are needed.
- **D3 — Identity = trained SDXL LoRA + FaceDetailer.** No face-swap, no InstantID in v1.
- **D4 — PWA (React + Vite) + FastAPI + SQLite.** Installable to the home screen; no app store.
- **D5 — Pose from reference = DWPose skeleton → SDXL OpenPose ControlNet.** (Pose composer deferred.)
- **D6 — Consent + watermark are build-time primitives.** C2PA deferred (personal use; low value now).
- **D7 — handoff.md + change_log.md stay separate from plan.md.**
- **D8 — Consent is a durable record** (`consents` table + stored evidence file), checked before training
  and before LoRA generation.
- **D9 — Hosting = small VPS (~$5/mo) + Cloudflare Tunnel + Cloudflare Access** (email one-time code, one
  allowed email). Requires a domain managed in Cloudflare.
- **D10 — Compute = RunPod Serverless** (gen + vision endpoints) **+ one RunPod GPU pod for training.**
  Models + LoRA live on a RunPod network volume; attaching it pins endpoints to that volume's datacenter —
  choose the datacenter once, for GPU availability.
- **D11 — Single user.** No account system beyond Cloudflare Access; the API trusts only the Access identity.
- **D12 — One-off scripted training.** kohya sd-scripts SDXL LoRA on a pod, config committed to `training/`.
  No in-app wizard in v1.
- **D13 — Acceptance = real round-trip with a named artifact; external signatures verified live.**
- **D14 — Photo→JSON runs on an open vision-language model on your own serverless worker** (Qwen-VL family or
  equivalent, permissive license; resolve the current model + license at build time). Photos don't go to a
  third-party AI API.
- **D15 — Explicit terms are REPLACED with neutral equivalents**, via a versioned mapping file + normalization
  (case, plurals, spacing, common obfuscations), plus a VLM instruction to describe neutrally. Replacements are
  shown to you. The neutralizer is applied to every outgoing prompt. A word list alone misses rephrasing, so
  the VLM instruction and the test corpus (Phase 5) are both required.
- **D16 — Serverless progress = job states, not live steps.** UI shows queued → running → done (+ elapsed time
  and a cold-start indicator); no per-step preview stream.
- **D17 — Worker images are built in GitHub Actions** and pushed to a container registry; the laptop isn't
  required to build or run anything.

Adding/altering a decision = a `change_log.md` entry tagged `[DECISION]` + an edit here.

---

## 6. Agent roster (roles Claude Code takes on; the manager role dispatches by phase)
Claude Code may run these as subagents or as sequential roles in one session. Parallel work only if `Owns`
don't overlap. Every role follows §2 and obeys §4. Logging and Git are cross-cutting: every role logs and
commits + pushes after each step on the current phase branch; the manager owns branch creation and merges (§13).

| Role | Owns | Phases | Must not |
|---|---|---|---|
| **manager** | control files, branches/merges | all | start a phase before predecessor PASS; merge without PASS |
| **infra-agent** | `infra/`, `.env.example`, Cloudflare/VPS/RunPod setup | 0, 8 | open inbound web ports; put secrets in git or the PWA |
| **worker-agent** | `worker/gen/`, `manifest.yaml`, CI image build, network volume | 1 | install outside manifest; set max workers > 1 or min workers > 0 |
| **backend-agent** | `server/` (api, runpod_client, queue), `server/workflows/` | 2–3 | run models on the VPS; bypass runpod_client; skip cost guard |
| **data-agent** | `server/data/`, `migrations/` | 4 | ship schema change without migration; hard-delete without trash |
| **vision-agent** | `worker/vision/`, `server/prompt/`, `server/filters/` | 5 | send photos to third-party AI APIs; let an un-neutralized prompt reach generation |
| **training-agent** | `training/` | 6 | train without a consent record; leave a pod running; leave training photos on the volume |
| **frontend-agent** | `web/` | 7 | store secrets client-side; reimplement backend logic in the client |
| **qa-agent** | `tests/`, CI | all gates | mark PASS on mocks or unverified signatures |

---

## 7. Phased plan (build in order; each phase = one branch)
Each phase: **Goal · Steps · Acceptance · Unlocks.** Before Phase 7, phases are exercised via API calls,
scripts, and `tests/` — acceptance names those artifacts.

### Phase 0 — Infrastructure & access (infra-agent)
**Goal:** a private URL only you can open, on a hardened VPS, with RunPod ready.
**Steps:** (0.0) materialize §9 + §11 files. (0.1) prerequisites checklist for the human: domain on
Cloudflare, VPS (Ubuntu LTS), RunPod account + API key, container registry. (0.2) `infra/setup_vps.sh`:
updates, firewall (SSH only, key-only, no password login), non-root user, Python, systemd units.
(0.3) Cloudflare Tunnel (`cloudflared`) → `https://studio.<domain>` → API on `127.0.0.1`. (0.4) Cloudflare
Access app: allow only your email (one-time code). (0.5) API serves a "hello" page and verifies the Access JWT.
(0.6) RunPod network volume created in the chosen datacenter.
**Acceptance:** phone opens the URL → email code → hello page (screenshot); a different email is denied
(screenshot); external port scan shows only SSH; a request straight to the API without a valid Access JWT
is rejected (test log).
**Unlocks:** everything is reachable only by you.

### Phase 1 — Generation worker (worker-agent)
**Goal:** a RunPod Serverless endpoint that runs any ComfyUI API workflow and returns images.
**Steps:** rewrite `manifest.yaml` to v3 scope (§9); `worker/gen/Dockerfile` based on the maintained RunPod
ComfyUI worker (resolve current repo + version); custom nodes pinned per manifest (controlnet_aux for DWPose,
Impact-Pack (+Subpack) for FaceDetailer, UltimateSDUpscale); volume-population script puts models on the
network volume (SDXL base + VAE, OpenPose ControlNet SDXL, face detector, upscaler) with sha256 checks;
GitHub Actions builds/pushes the image; endpoint config: max workers 1, min 0, network volume attached; dump
`/object_info` from the running worker to `server/workflows/object_info.json`.
**Acceptance:** `/runsync` with a bundled SDXL txt2img workflow returns an image (saved file); cold-start and
warm timings recorded in change_log; endpoint shows 0 workers when idle (screenshot).
**Unlocks:** GPU generation without your laptop.

### Phase 2 — Backend core + RunPod client + job queue (backend-agent)
**Goal:** the VPS can submit, track, cancel, and collect jobs reliably and within budget.
**Steps:** FastAPI app; `runpod_client.py` (submit `/run`, poll `/status`, `/cancel`, retries with backoff,
timeouts, download outputs to VPS storage); persistent SQLite job queue; cost guard (daily cap, monthly
ceiling, per-job cost estimate from duration); structured logs.
**Acceptance:** API call → job → image stored on the VPS (file); cancel mid-job works (log); restart the VPS
service mid-job → job resumes polling and completes (log); exceeding the daily cap is refused (test).
**Unlocks:** reliable job lifecycle.

### Phase 3 — Workflow templates (backend-agent)
**Goal:** every Generate-tab feature is a validated template.
**Steps:** API-format templates: `txt2img_lora`, `img2img_lora`, `pose_ref` (DWPose → OpenPose ControlNet +
LoRA), `face_fix` (FaceDetailer, optional on any of the above), `upscale`; `builder.py` injects params by node
id and validates against `object_info.json`; `registry.py` intent→template; invisible watermark applied to
every saved output on the VPS. (Before the LoRA exists, templates are tested with LoRA disabled.)
**Acceptance:** each template produces an image via the API (one saved file per template); invalid params fail
fast against `object_info.json`; watermark is detectable on outputs (test).
**Unlocks:** the whole Generate tab, minus UI.

### Phase 4 — Data layer (data-agent)
**Goal:** gallery, prompts, LoRA, and consent persist and survive change.
**Steps:** SQLite tables: `generations` (intent, params, seed, prompt, source extract id, parent_id, asset ids,
cost, status), `assets`, `prompt_extracts` (photo, JSON, replacements made), `loras` (name, trigger, base_arch,
file), `consents` (subject, scope, date, evidence file); versioned `migrations/`; trash (soft delete,
recoverable); gallery query API (paginated, newest first, favorites filter); nightly backup of DB + gallery
(`infra/backup.sh`) to a second location.
**Acceptance:** 10 generations → paginated gallery query returns them with params; delete → restore from trash;
migration upgrades an older DB without loss; backup restored into a clean DB matches (test).
**Unlocks:** Gallery tab data; consent has a home.

### Phase 5 — Photo → JSON prompt (vision-agent)
**Goal:** a photo becomes clean structured JSON and a ready SDXL prompt.
**Steps:** `worker/vision/` serverless endpoint with an open VLM (D14); fixed **JSON schema**:
`{subject, appearance, outfit, pose, expression, scene, lighting, camera:{shot, angle, lens}, style, colors,
quality_tags[], negative[]}`; VLM instructed to return only schema-valid JSON and describe neutrally; server
validates with Pydantic (one auto-retry on invalid JSON); **neutralizer** (D15) over every field using
`neutral_map.yaml` with normalization → returns cleaned JSON + list of replacements; `compile_prompt(json, lora)`
→ SDXL positive/negative prompt with the trigger word inserted; optional "use this photo's pose" flag that passes
the photo to `pose_ref`.
**Acceptance:** 10 varied test photos → 10/10 schema-valid JSON files; a test corpus of explicit terms and
obfuscated variants → zero survive the neutralizer (test report); replacements list is returned; compiled prompt
for one photo generates an image via Phase 3 (file).
**Unlocks:** the Prompt tab, minus UI.

### Phase 6 — One-off LoRA training (training-agent)
**Goal:** one high-quality SDXL LoRA of the consented subject, installed on the network volume.
**Steps:** consent record created and verified first (refuse otherwise); `training/prepare.py`: dedup
(perceptual hash), face-centered crops, resolution bucketing, captions (auto + editable, trigger word);
`training/config.toml` for kohya sd-scripts SDXL LoRA (resolve current flags from the pinned version);
`training/run_pod.sh`: start GPU pod with the network volume → train → fixed-seed sample grid every N steps →
copy chosen checkpoint to the volume → delete training photos from volume/pod → terminate pod; register the
LoRA in `loras`.
**Acceptance:** training without a consent row is refused (log); a full run produces the LoRA file + sample grid
(files); pod is terminated afterwards (RunPod screenshot); `txt2img_lora` with the trigger produces a recognizable
likeness (file, human-judged); training photos absent from the volume (listing).
**Unlocks:** the identity half of the product.

### Phase 7 — The PWA (frontend-agent)
**Goal:** the three-tab phone app over the proven API.
**Steps:** React + Vite PWA (manifest, icons, service worker for app-shell caching; installable).
- **Prompt tab:** camera/upload → progress → editable JSON fields + raw JSON view → highlighted replacements →
  "Send to Generate".
- **Generate tab:** prompt (prefilled from Prompt tab or typed), LoRA on/off + strength, mode (text→image /
  image→image + strength / pose from reference photo), face fix toggle, upscale toggle, size/seed/steps
  (advanced, collapsed), job status (queued/running/done + elapsed + cold-start note), cancel.
- **Gallery tab:** grid → detail (image, prompt, JSON, settings) → reuse settings, favorite, download/share,
  delete (trash).
- **Settings:** jobs today, spend so far vs caps, consent record view, about.
Mobile-first layout; no secrets in the client; all calls same-origin through Cloudflare Access.
**Acceptance (on your phone, laptop off):** install to home screen → photo → JSON with replacements → send to
Generate → pose-from-reference + face fix + upscale → image appears in Gallery → reuse settings works
(screen recording).
**Unlocks:** the shippable app.

### Phase 8 — Hardening (infra-agent + qa-agent)
**Goal:** it stays private, cheap, and recoverable.
**Steps:** backup restore drill; dependency + OS update routine; log rotation; alert (email/Telegram) when
spend reaches 80% of the monthly ceiling; runbook (§12) verified end to end.
**Acceptance:** restore drill from backup succeeds (log); spend alert fires in a test (log); §10 matrix all PASS.

---

## 8. Interface contracts (keep stable — `server/api/contracts.py`, Pydantic)
- `POST /prompt/extract {photo} -> {extract_id, json, replacements[], compiled_prompt}`
- `POST /jobs {intent, params} -> {job_id}` — intents: `txt2img_lora | img2img_lora | pose_ref | upscale`;
  `params.face_fix: bool`, `params.upscale: bool`
- `GET /jobs/{id} -> {state: queued|running|done|failed|cancelled, elapsed_s, cold_start: bool, asset_ids[], error?}`
- `POST /jobs/{id}/cancel -> {ok}`
- `GET /gallery?cursor&favorites -> {items[], next_cursor}`; `GET /generations/{id}`;
  `POST /generations/{id}/favorite`; `DELETE /generations/{id}` (trash); `POST /generations/{id}/restore`
- `Lora = {id, name, trigger, base_arch, file}` — base_arch mandatory
- `consent(subject) -> bool` — training and LoRA generation call this; false → refuse
- `GET /usage -> {jobs_today, daily_cap, spend_month_est, monthly_ceiling}`

Contract change = `[DECISION]` entry + update here, then notify dependent roles.

---

## 9. On-disk config files (written verbatim in step 0.0)

**`.gitignore`**
```
.env
/var/
/models/
*.safetensors
*.ckpt
*.pt
*.pth
*.onnx
__pycache__/
.venv/
node_modules/
/web/dist/
/training/dataset/
/training/output/
*.db
```

**`.env.example`**
```
# Copy to .env ON THE VPS ONLY. Never commit .env. Never expose these to the PWA.
RUNPOD_API_KEY=
RUNPOD_GEN_ENDPOINT_ID=
RUNPOD_VISION_ENDPOINT_ID=
RUNPOD_NETWORK_VOLUME_ID=
HF_TOKEN=                      # if any pinned model is gated
CF_ACCESS_TEAM_DOMAIN=         # e.g. yourteam.cloudflareaccess.com
CF_ACCESS_AUD=                 # Access application audience tag (JWT verification)
API_HOST=127.0.0.1
API_PORT=8000
DATA_DIR=./var
LOG_DIR=./var/logs
DAILY_JOB_CAP=100
MONTHLY_SPEND_CEILING_USD=30
```

**`manifest.yaml`** — rewritten in Phase 1 to the v3 scope: keep SDXL base + VAE, controlnet_aux (DWPose),
Impact-Pack (+Subpack) + face detector, UltimateSDUpscale, kohya sd-scripts; **add** SDXL OpenPose ControlNet,
upscaler model, the VLM for `worker/vision/`; **remove** FLUX.2, Z-Image, Qwen-Image, InstantID/InsightFace,
IP-Adapter, IC-Light, SAM2, ai-toolkit, OneTrainer. Keep per-model `license` flags; the face detector is AGPL
(flag it). Pin repos/commits/files/sha256 against the live sources.

---

## 10. Verification matrix (qa-agent; runnable in `tests/`; each produces a named artifact)
| Gate | Pass condition |
|---|---|
| P0 | only your email gets in; other email denied; only SSH open; API rejects missing/invalid Access JWT |
| P1 | `/runsync` txt2img image returned; idle endpoint at 0 workers; cold/warm timings logged |
| P2 | job round-trip to VPS storage; cancel works; survives service restart; daily cap enforced |
| P3 | each template outputs an image; bad params rejected vs `object_info.json`; watermark detectable |
| P4 | gallery pagination; trash/restore; migration lossless; backup restore matches |
| P5 | 10/10 schema-valid JSON; zero explicit terms survive the corpus; compiled prompt generates |
| P6 | consent-less training refused; LoRA + sample grid produced; pod terminated; photos removed |
| P7 | full phone flow, laptop off, installed PWA (screen recording) |
| P8 | restore drill OK; spend alert fires; all prior gates still PASS |

---

## 11. Seeds for the living files (written in step 0.0, then live separately)

**`handoff.md` (seed)**
```
# HANDOFF — read FIRST, rewrite LAST
## Current state
- Phase: 0 — Infrastructure & access
- Step: 0.0 — Materialize on-disk files (.gitignore, .env.example, handoff.md, change_log.md)
- Status: NOT STARTED
- Owner role: infra-agent
- Branch: phase-0-infra
- Last updated: 0000-00-00 00:00 by manager
## Next action (do exactly this)
Create branch `phase-0-infra` off `main`, write the files from plan.md §9 and §11 to the repo root,
then produce the 0.1 prerequisites checklist for the human (domain on Cloudflare, VPS, RunPod key, registry).
## Acceptance for this step
All §9 + §11 files exist at repo root and parse.
## Blockers
- Human must provide: domain in Cloudflare, VPS access, RunPod API key.
## Notes
- No phase starts before its predecessor has a PASS in change_log.md.
- Every acceptance check produces a named artifact; no mocks, no assumed API signatures.
- After each step: commit + push to the phase branch; merge to `main` only on PASS (§13).
```

**`change_log.md` (seed)**
```
# CHANGE LOG — append-only, newest first
# Format:
# ## <YYYY-MM-DD HH:MM> — [Phase X.Y] <title>
# - Role / Branch / Changed / Files / Artifact / Why / Verification(PASS|FAIL) / Follow-ups
---
## 0000-00-00 00:00 — [Phase 0.0] Repository initialized
- Role: manager
- Branch: phase-0-infra (off main)
- Changed: scaffolded repo + control protocol from plan.md v3
- Files: /plan.md, /CLAUDE.md, /handoff.md, /change_log.md
- Artifact: control files present at repo root
- Why: lock the operating protocol before building
- Verification: PASS — control files present
- Follow-ups: infra-agent → §9 files, prerequisites checklist, VPS + Cloudflare setup
```

---

## 12. Usage runbook (for you, after the build)
- **Open the app:** phone → `https://studio.<domain>` → email code → "Add to Home Screen" once; afterwards
  open it like any app. Access sessions expire periodically; you'll re-enter an email code.
- **Photo → prompt:** Prompt tab → shoot/upload → check JSON fields and the highlighted replacements → edit if
  needed → Send to Generate.
- **Generate:** pick mode (text / image-to-image / pose from reference) → LoRA on → face fix + upscale as
  needed → Generate. The first job after idle takes longer (cold start); later ones are faster.
- **Gallery:** tap an image for its prompt/JSON/settings → reuse, favorite, download, or delete (recoverable
  from trash).
- **Costs:** Settings shows jobs today and estimated monthly spend; caps stop jobs before overspending.
  Idle cost ≈ VPS + network-volume storage.
- **Retraining:** only via `training/run_pod.sh`, after updating the consent record if scope changes.

---

## 13. Git / version-control workflow (GitHub repo `salahsterr/imageo`; mandatory)
**Branching model**
- `main` only receives **phase-complete, acceptance-PASS** work (plus this initial constitution commit).
- One branch per phase, `phase-<N>-<slug>`: `phase-0-infra`, `phase-1-gen-worker`, `phase-2-backend`,
  `phase-3-workflows`, `phase-4-data`, `phase-5-prompt-vision`, `phase-6-lora-training`, `phase-7-pwa`,
  `phase-8-hardening`.
- The manager creates each branch off the latest `main` and merges it back at phase end; the next phase
  branches off the updated `main`.

**Per step (§2.d)**
- `git add -A && git commit -m "[Phase X.Y] <title> — <PASS|FAIL>" && git push` on the phase branch.
- Commit message mirrors the change_log header. Push every step (safety net if a session dies).
- Source/config only — models, `var/`, datasets, DB files, and secrets stay gitignored (§9).

**Merge (phase finished)**
- Only after the phase gate (§10) has a PASS in `change_log.md`. No PASS, no merge.
- Via a **PR into `main`** with a merge commit (no fast-forward); optionally tag `phase-<N>-pass`.
- CI (GitHub Actions) must be green on the PR; worker images are built from `main` (D17).

---

## 14. Non-goals / deferred (v1 scope boundary — expanding scope needs a [DECISION])
**Deferred to v2:** editor ops (inpaint/outpaint/instruction edit/SAM2 masks), pose composer (multi-photo pose
blending), dataset health score, in-app training wizard, C2PA credentials, additional base models (FLUX.2,
Z-Image), multi-subject LoRA management UI.
**Not planned:** video/animation; multiple users or sharing; public access; native app-store apps; commercial
use (personal use only — revisit model licenses before changing this); rebuilding diffusion/training internals;
running models on the VPS.
