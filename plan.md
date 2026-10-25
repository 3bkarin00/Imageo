# plan.md — Personal AI Image Studio "imageo" (mobile PWA + cloud GPU) · v3.2

**This is the project constitution for building `imageo` with Claude Code** (repo:
`github.com/3bkarin00/Imageo`). It is self-contained: operating protocol, rules, decisions, agent
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
- **v3.1 [DECISION] — VPS replaced by Cloudflare serverless** (Workers + D1 + R2, ~$0/month, pay-per-use).
  Backend is a TypeScript Worker; job completion arrives by **RunPod webhook** + a reconcile cron instead of a
  long-running poller; the **GPU worker watermarks and uploads images straight to R2** (Workers' 10 ms CPU limit
  rules out image work in the Worker). Touches §1, §3, §4, D4, D9, D16, D18, Phases 0/1/2/3/4/8, §8–§10, §12.
- **v3.2 [DECISION] — invisible watermark removed.** The subject is a co-creator of the photoshoots, not a third
  party, so a hidden "AI-made" mark adds no protection she wants. Generation metadata stays in D1. (D6 revised.)

---

## 1. North Star
A **private, single-user** image studio you use **from your phone**, anywhere:

1. **Prompt tab** — take/upload a photo → a vision model describes it as **structured JSON** → any
   explicit terms are **replaced with neutral equivalents** (shown to you) → one tap sends it to Generate.
2. **Generate tab** — text→image with your subject's LoRA, plus: **pose from a reference photo**,
   **image-to-image**, **face fix**, **upscale**.
3. **Gallery tab** — every generation with its prompt/JSON/settings; reuse, favorite, download, delete.

**We do not build a diffusion engine — we orchestrate ComfyUI**, running on RunPod Serverless.
Codenames: **`studio`** = the app we build (PWA + Cloudflare Worker backend). **`worker`** = the RunPod GPU side.

**System shape**
```
Phone (PWA) ──HTTPS──▶ Cloudflare Access (email one-time code, only you)
                         ▼
        Cloudflare Worker on studio.<domain> (free tier, pay-per-use)
          • serves the PWA (static assets) + JSON API
          • D1 = database (jobs, gallery, consent)   • R2 = image storage
          • RunPod API key lives ONLY here (Worker secret)
                         │  /run with webhook ──▶        ◀── webhook on completion (/hooks/runpod, HMAC-signed)
                         ▼
        RunPod Serverless endpoints ── network volume (models, LoRA)
          • gen    : ComfyUI worker (SDXL, ControlNet, FaceDetailer, upscaler) → upload to R2
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
| `.gitignore`, `.dev.vars.example` | Repo hygiene, declared Worker secrets/vars (§9) | Static |
| `wrangler.toml` | Worker config: D1 + R2 bindings, cron, custom domain, `workers_dev = false` | Phase 0 |
| `manifest.yaml` | Pinned worker deps, custom nodes, models (+license flags) | Rewritten Phase 1; bump via decision |
| `worker/gen/` | Dockerfile + handler for the ComfyUI serverless worker | Built Phase 1 |
| `worker/vision/` | Dockerfile + handler for the VLM serverless worker | Built Phase 5 |
| `training/` | Dataset prep + kohya config + run script for the one-off pod | Built Phase 6 |
| `server/` | Cloudflare Worker backend (TypeScript, Hono): api, runpod client, jobs, data, prompt service | Phases 0, 2–5 |
| `server/workflows/templates/` | ComfyUI API-format workflow templates | Phase 3 |
| `server/filters/neutral_map.yaml` | Explicit term → neutral term mapping | Phase 5 |
| `migrations/` | Versioned D1 migrations (`wrangler d1 migrations`) | Grows with schema |
| `web/` | PWA (React + Vite), 3 tabs | Phase 7 |
| `infra/` | Cloudflare/RunPod setup notes, Access policy record, backup scripts | Phases 0, 8 |
| `.github/workflows/` | CI: tests, Worker deploy (wrangler), build/push GPU worker images, scheduled D1 backup | Phases 0, 1, 4 |
| `tests/` | Runnable gate checks (§10) | All phases |

---

## 4. RULES (hard constraints — violating these is a failed task)
**Architecture**
- studio NEVER implements diffusion/sampling/training loops. Generation = ComfyUI on the worker;
  training = pinned kohya sd-scripts on a pod; JSON description = an open VLM on the vision worker.
- **The Worker never runs models or heavy image/byte work.** It orchestrates, stores metadata, and serves.
  Free-plan Workers get 10 ms CPU per request: image decoding, encoding and base64 handling happen on the
  GPU worker; the Worker only streams R2 objects and handles JSON.
- All RunPod calls go through `server/src/runpod_client.ts`. No raw RunPod HTTP elsewhere.
- All generation graphs are built by `server/src/workflows/builder.ts` from template JSON + params,
  validated against an `/object_info` snapshot taken from the deployed worker image.
- `manifest.yaml` is the single source of truth for worker deps/nodes/models. Nothing installed or
  downloaded outside the worker Dockerfiles / volume-population script.

**Security (single user)**
- The PWA and API are reachable **only through Cloudflare Access**, allowing exactly one email (yours).
- The Worker is reachable **only on `studio.<domain>`**: `workers_dev = false` and preview URLs disabled, so no
  `*.workers.dev` address bypasses Access.
- **Secrets live only as Worker secrets** (`wrangler secret put`), RunPod endpoint env vars, and GitHub Actions
  secrets where CI needs them. Never in the PWA, never in git.
- The Worker verifies the Cloudflare Access JWT on every request except `/hooks/runpod/*` (defense if Access
  is misconfigured).
- `/hooks/runpod/*` is the only Access-bypassed path. It accepts a call only with a valid per-job HMAC
  signature, for a job that exists and is still open; anything else → 401, logged.
- The R2 key given to the GPU worker can write only to the `outputs/` prefix of the bucket (scoped token).

**Cost guard (serverless is billed per use)**
- Serverless endpoints: **min/active workers = 0**, **max workers = 1**, short idle timeout.
- The backend enforces a **daily job cap** and a **monthly spend ceiling** (configurable); over the cap,
  jobs are refused with a clear message.
- The training pod is started by script and **auto-terminated** when training ends or fails.

**Operational**
- Jobs are recorded **persistently** in D1 before submission; nothing lives only in Worker memory. Every job
  can be cancelled.
- Completion arrives by **RunPod webhook**; a **cron trigger (≤ every 10 min)** reconciles jobs still open past
  their expected time via `/status`, because RunPod keeps async results only ~30 minutes after completion.
- Every job is logged (structured, to a D1 `job_events` table): job id, intent, params, RunPod request id,
  status, error, cost estimate.
- The GPU worker writes outputs **directly to R2** and returns only object keys; the gallery never depends on
  RunPod retaining outputs.

**Consent / trust**
- The subject is a real person: a **consent record** (her written consent, date, scope) must exist
  before training and before any generation using her LoRA (D8).
- Training photos are deleted from the network volume and pod after the LoRA is produced; originals are
  kept (outside R2 and git) only if you choose to.
- Every output has its generation metadata (prompt, settings, seed, LoRA) recorded in D1. No watermark (D6).
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
- **D4 — PWA (React + Vite) + Cloudflare Worker (TypeScript, Hono) + D1 + R2.** Installable to the home screen;
  no app store. The PWA is served as the Worker's static assets, so app and API share one origin.
- **D5 — Pose from reference = DWPose skeleton → SDXL OpenPose ControlNet.** (Pose composer deferred.)
- **D6 — Consent is a build-time primitive; no watermark, no C2PA.** The subject co-creates the photoshoots for
  her own use, so hidden provenance marks add no value; outputs stay untouched lossless PNGs. (Watermark removed v3.2.)
- **D7 — handoff.md + change_log.md stay separate from plan.md.**
- **D8 — Consent is a durable record** (`consents` table + stored evidence file), checked before training
  and before LoRA generation.
- **D9 — Hosting = Cloudflare serverless (Workers + D1 + R2) behind Cloudflare Access** (email one-time code,
  one allowed email), on the free tier — pay-per-use, ~$0 idle. Requires a domain managed in Cloudflare.
  (v3 used a VPS; replaced in v3.1 because a VPS is billed for every hour it exists.)
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
  and a cold-start indicator); no per-step preview stream. The PWA polls the Worker's `GET /jobs/{id}`; the
  Worker learns completion from the RunPod webhook (D18), never by holding a connection open.
- **D18 — Job completion = RunPod webhook + reconcile cron.** `/run` is called with a `webhook` URL carrying a
  per-job HMAC; RunPod retries a failed webhook twice. A cron trigger catches anything missed.
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
| **infra-agent** | `infra/`, `wrangler.toml`, `.dev.vars.example`, Cloudflare/RunPod setup | 0, 8 | leave `workers.dev`/preview URLs on; Access-bypass anything but `/hooks/runpod/*`; put secrets in git or the PWA |
| **worker-agent** | `worker/gen/`, `manifest.yaml`, CI image build, network volume | 1 | install outside manifest; set max workers > 1 or min workers > 0 |
| **backend-agent** | `server/` (api, runpod_client, jobs, hooks, cron), `server/src/workflows/` | 2–3 | run models or image/byte work in the Worker; bypass runpod_client; skip cost guard; accept an unsigned webhook |
| **data-agent** | `server/src/data/`, `migrations/` | 4 | ship schema change without migration; hard-delete without trash |
| **vision-agent** | `worker/vision/`, `server/src/prompt/`, `server/filters/` | 5 | send photos to third-party AI APIs; let an un-neutralized prompt reach generation |
| **training-agent** | `training/` | 6 | train without a consent record; leave a pod running; leave training photos on the volume |
| **frontend-agent** | `web/` | 7 | store secrets client-side; reimplement backend logic in the client |
| **qa-agent** | `tests/`, CI | all gates | mark PASS on mocks or unverified signatures |

---

## 7. Phased plan (build in order; each phase = one branch)
Each phase: **Goal · Steps · Acceptance · Unlocks.** Before Phase 7, phases are exercised via API calls,
scripts, and `tests/` — acceptance names those artifacts.

### Phase 0 — Infrastructure & access (infra-agent)
**Goal:** a private URL only you can open, served by a Cloudflare Worker, with D1, R2 and RunPod ready.
**Steps:** (0.0) materialize §9 + §11 files. (0.1) prerequisites checklist for the human: domain on
Cloudflare, Zero Trust team, R2 enabled, RunPod account + API key, the one allowed email. (0.2) Worker skeleton
(`server/`, TypeScript + Hono) + `wrangler.toml`: D1 database and R2 bucket bindings, custom domain
`studio.<domain>`, `workers_dev = false`, preview URLs off; serves a "hello" page. (0.3) Cloudflare Access app on
`studio.<domain>`: allow only your email (one-time code); a bypass policy for `/hooks/runpod/*` only, recorded in
`infra/access_policy.md`. (0.4) Worker middleware verifies the Access JWT (team domain + AUD) on every non-hook
route; the hook route checks the HMAC and returns 401 otherwise. (0.5) GitHub Actions deploys the Worker with a
scoped Cloudflare API token secret. (0.6) RunPod network volume created in the chosen datacenter.
**Acceptance:** phone opens the URL → email code → hello page (screenshot); a different email is denied
(screenshot); the `*.workers.dev` address does not serve the app (test log); a request with a forged or missing
Access JWT is rejected (test log); `/hooks/runpod/x` without a valid signature → 401 (test log).
**Unlocks:** everything is reachable only by you.

### Phase 1 — Generation worker (worker-agent)
**Goal:** a RunPod Serverless endpoint that runs any ComfyUI API workflow and returns images.
**Steps:** rewrite `manifest.yaml` to v3 scope (§9); `worker/gen/Dockerfile` based on the maintained RunPod
ComfyUI worker (resolve current repo + version); custom nodes pinned per manifest (controlnet_aux for DWPose,
Impact-Pack (+Subpack) for FaceDetailer, UltimateSDUpscale); volume-population script puts models on the
network volume (SDXL base + VAE, OpenPose ControlNet SDXL, face detector, upscaler) with sha256 checks;
handler post-processing: **upload the lossless PNG to R2 `outputs/` via the S3-compatible API → return
only object keys** (verify the base worker's S3 env vars against its current docs; if it can't target R2, add
the upload in our handler); GitHub Actions builds/pushes the image; endpoint config: max workers 1, min 0,
network volume attached, R2 write-scoped key in endpoint env; dump `/object_info` from the running worker to
`server/src/workflows/object_info.json`.
**Acceptance:** `/runsync` with a bundled SDXL txt2img workflow returns an R2 object key, and that object is a
valid PNG byte-identical to what ComfyUI produced (downloaded file + hash check); cold-start and warm timings recorded in change_log;
endpoint shows 0 workers when idle (screenshot).
**Unlocks:** GPU generation without your laptop.

### Phase 2 — Backend core + RunPod client + job queue (backend-agent)
**Goal:** the Worker can submit, track, cancel, and collect jobs reliably and within budget.
**Steps:** `runpod_client.ts` (submit `/run` with a per-job signed `webhook` URL, `/status/{id}`, `/cancel`,
retries with backoff, timeouts — verify every path and payload against current RunPod docs); D1 `jobs` +
`job_events` tables (job row written before submission); `POST /hooks/runpod/{job_id}` (HMAC check → mark done,
record R2 keys and duration); reconcile cron (≤ every 10 min: open jobs past expected time → `/status` → settle);
cost guard (daily cap, monthly ceiling, per-job cost estimate from reported duration); structured events.
**Acceptance:** API call → job → webhook → D1 row done with an R2 key that downloads (file); cancel mid-job works
(log); a job whose webhook is blocked on purpose is settled by the cron (log); exceeding the daily cap is refused
(test); a replayed or forged webhook is rejected (test).
**Unlocks:** reliable job lifecycle.

### Phase 3 — Workflow templates (backend-agent)
**Goal:** every Generate-tab feature is a validated template.
**Steps:** API-format templates: `txt2img_lora`, `img2img_lora`, `pose_ref` (DWPose → OpenPose ControlNet +
LoRA), `face_fix` (FaceDetailer, optional on any of the above), `upscale`; `builder.ts` injects params by node
id and validates against `object_info.json`; `registry.ts` intent→template; input images (img2img source, pose
reference) are stored in R2 and passed to RunPod as short-lived signed URLs, never as base64 through the Worker.
(Before the LoRA exists, templates are tested with LoRA disabled.)
**Acceptance:** each template produces an image via the API (one saved file per template); invalid params fail
fast against `object_info.json`; inputs reach RunPod as signed URLs (test).
**Unlocks:** the whole Generate tab, minus UI.

### Phase 4 — Data layer (data-agent)
**Goal:** gallery, prompts, LoRA, and consent persist and survive change.
**Steps:** D1 tables: `generations` (intent, params, seed, prompt, source extract id, parent_id, asset ids,
cost, status), `assets` (R2 key, size, hash), `prompt_extracts` (photo key, JSON, replacements made), `loras`
(name, trigger, base_arch, file), `consents` (subject, scope, date, evidence file key); versioned `migrations/`;
trash (soft delete, recoverable; R2 objects removed only when trash is emptied); gallery query API (paginated,
newest first, favorites filter); images served by streaming from R2 through the Worker (behind Access);
scheduled GitHub Action exports D1 (`wrangler d1 export` — verify flags) to a private `backups/` prefix; D1's
built-in point-in-time restore noted as the first recovery option.
**Acceptance:** 10 generations → paginated gallery query returns them with params; delete → restore from trash;
migration upgrades an older DB without loss; backup restored into a clean DB matches (test).
**Unlocks:** Gallery tab data; consent has a home.

### Phase 5 — Photo → JSON prompt (vision-agent)
**Goal:** a photo becomes clean structured JSON and a ready SDXL prompt.
**Steps:** `worker/vision/` serverless endpoint with an open VLM (D14); fixed **JSON schema**:
`{subject, appearance, outfit, pose, expression, scene, lighting, camera:{shot, angle, lens}, style, colors,
quality_tags[], negative[]}`; VLM instructed to return only schema-valid JSON and describe neutrally; the photo
goes to R2 first and reaches the VLM as a short-lived signed URL; the Worker validates the JSON with zod (one
auto-retry on invalid JSON); **neutralizer** (D15) over every field using
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
**Steps:** React + Vite PWA (manifest, icons, service worker for app-shell caching; installable), built into
the Worker's static assets.
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
**Steps:** backup restore drill (D1 export → fresh database); dependency update routine (Dependabot on the repo);
`job_events` retention/cleanup; alert (email/Telegram) when spend reaches 80% of the monthly ceiling, and when
Cloudflare free-tier usage (requests, D1, R2) passes 80%; runbook (§12) verified end to end.
**Acceptance:** restore drill from backup succeeds (log); spend alert fires in a test (log); §10 matrix all PASS.

---

## 8. Interface contracts (keep stable — `server/src/contracts.ts`, zod)
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
- `GET /assets/{id}` — streams the image from R2 (behind Access)
- `POST /hooks/runpod/{job_id}?sig=<hmac>` — RunPod callback; the only Access-bypassed route; returns 200 fast

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
.dev.vars
.wrangler/
```

**`.dev.vars.example`**
```
# Local dev only: copy to .dev.vars (gitignored). In production set each SECRET with `wrangler secret put <NAME>`.
# Plain vars (caps, IDs) go in wrangler.toml [vars]. Never expose any of these to the PWA.
# --- secrets ---
RUNPOD_API_KEY=
WEBHOOK_HMAC_SECRET=           # random 32+ bytes; signs per-job webhook URLs
R2_SIGNING_ACCESS_KEY_ID=      # R2 S3 key used to mint short-lived signed URLs for RunPod
R2_SIGNING_SECRET_ACCESS_KEY=
# --- vars ---
RUNPOD_GEN_ENDPOINT_ID=
RUNPOD_VISION_ENDPOINT_ID=
CF_ACCESS_TEAM_DOMAIN=         # e.g. yourteam.cloudflareaccess.com
CF_ACCESS_AUD=                 # Access application audience tag (JWT verification)
PUBLIC_BASE_URL=               # https://studio.<domain>
DAILY_JOB_CAP=100
MONTHLY_SPEND_CEILING_USD=30
# Set on the RunPod endpoints instead (not here): R2 write-scoped key for outputs/, HF_TOKEN if a model is gated.
# Set in GitHub Actions secrets: CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, registry credentials.
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
| P0 | only your email gets in; other email denied; `workers.dev` doesn't serve; forged/missing JWT rejected; unsigned hook → 401 |
| P1 | `/runsync` txt2img → lossless PNG in R2 (hash matches); idle endpoint at 0 workers; cold/warm timings logged |
| P2 | job → webhook → D1 done + R2 key; cancel works; blocked webhook settled by cron; daily cap enforced; forged webhook rejected |
| P3 | each template outputs an image; bad params rejected vs `object_info.json`; inputs passed as signed URLs |
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
- Step: 0.0 — Materialize on-disk files (.gitignore, .dev.vars.example, handoff.md, change_log.md)
- Status: NOT STARTED
- Owner role: infra-agent
- Branch: phase-0-infra
- Last updated: 0000-00-00 00:00 by manager
## Next action (do exactly this)
Create branch `phase-0-infra` off `main`, write the files from plan.md §9 and §11 to the repo root,
then produce the 0.1 prerequisites checklist for the human (domain on Cloudflare, Zero Trust, R2, RunPod key).
## Acceptance for this step
All §9 + §11 files exist at repo root and parse.
## Blockers
- Human must provide: domain in Cloudflare, Zero Trust team, R2 enabled, RunPod API key.
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
- Follow-ups: infra-agent → §9 files, prerequisites checklist, Cloudflare Worker + Access setup
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
  Idle cost ≈ RunPod network-volume storage + the domain; Cloudflare stays on the free tier for one user.
- **Retraining:** only via `training/run_pod.sh`, after updating the consent record if scope changes.

---

## 13. Git / version-control workflow (GitHub repo `3bkarin00/Imageo`; mandatory)
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
running models or image processing in the Worker; a VPS or any always-on server.
