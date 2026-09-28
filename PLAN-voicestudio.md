# PLAN-voicestudio.md — VoiceStudio on Olares: port, operation & upgrade playbook

Chart: `charts/voicestudio` · Olares baseline: **1.12.6+** (`apiVersion: v3`, `olaresManifest.version: 0.12.0`) · Upstream: https://github.com/debpalash/VoiceStudio (AGPL-3.0)

---

## 1. What this repo is

A community Olares port of **VoiceStudio's headless web-server image** (not the desktop app — the web UI + FastAPI backend on port 3900). It is designed for self-serve distribution:

- Anyone with an Olares can install it from this repo's GitHub Releases (chart `.tgz`) without touching `beclab/apps`.
- The chart carries full market-page metadata (description, screenshots, categories, license, docs, i18n en-US/zh-CN).
- Entrance is **public** (`authLevel: public`) so native desktop apps can sync from outside the Olares login wall; **every API rule still requires the instance's `OMNIVOICE_API_KEY`**, so the app effectively gates itself.

## 2. Current deployed state (reference)

| Item | Value |
|---|---|
| Olares instance | `my@cgtale.com` (Owner), OS 1.12.6, node `olares` amd64, NVIDIA GPU (CUDA 13.2, driver 595.84) |
| Installed chart | `voicestudio` **1.0.1** — upstream **0.5.6**, image `palashdeb/omnivoice-studio:0.5.6` |
| Compute mode | `nvidia` (HAMi vGPU slice: 6 Gi floor / 16 Gi cap), GPU pool sees 23.3 GB free |
| Entrance URL | `https://2ddf14ec.my.cgtale.com` (derived host: `md5("voicestudio")[:8].<zone>` — read it via `olares-cli settings apps list`) |
| Credentials | `OMNIVOICE_API_KEY` requested at install time (password field); travels through the chart Secret `voicestudio-voicesturo` → see `templates/secret.yaml`. Owner has the value; regenerate freely (change envs → upgrade) |
| Storage | `appData` → `/app/omnivoice_data` (DB, voices, projects, cuDNN side-load); `appCommon` → `Common/huggingface` (cross-app model cache) |
| Data sync | Native desktop app / OpenAI clients: `base_url = https://<entrance-host>` (`/v1`) + API key |

## 3. Architecture decisions (the load-bearing quirks)

Each of these was a real incident; do not "simplify" them away on the next pass:

1. **uid 1000 with no passwd entry.** Upstream image runs uvicorn as root by default; Olares policy requires uid 1000. Beyond `securityContext`, the *identity* envs (`USER`, `LOGNAME`, `LNAME`, `USERNAME`, `HOME=/tmp`) exist because `torch.compile`'s inductor calls `getpass.getuser()`/`pwd.getpwuid(1000)` and **crashes with `KeyError: 'getpwuid(): uid not found: 1000'`** without them (app then silently downgrades to eager mode). Symptom check: log line `torch.compile deleted for this session`.
2. **Probes are `exec curl -fsS http://127.0.0.1:3900/health`** — NOT `httpGet`. Kubelet HTTP probes transit the Olares envoy sidecar, which intermittently answered **403** (its RBAC/authz layer) while the app itself returned 200s for its whole uptime — the kubelet then liveness-killed a healthy container mid-generate (front-origin 403's, 15:18–15:30 UTC window). Exec probes run inside the container and bypass the sidecar entirely. Generous thresholds (`24 × 30s` liveness) ride out model-load stalls.
3. **cuDNN 8 side-load.** The CUDA image ships torch with cuDNN 9; CTranslate2 (under **WhisperX / Faster-Whisper**) needs cuDNN 8 and **calls `__fastfail` — killing the whole backend** when it's absent (`#1371`). `init-cudnn8` pip-installs `nvidia-cudnn-cu12==8.9.7.29` into `appData/cudnn8` (idempotent, survives pod restarts) and `LD_LIBRARY_PATH` exposes it. cuDNN8/cuDNN9 sonames don't collide (`.so.8` vs `.so.9`).
4. **`OMNIVOICE_ALLOWED_ORIGINS`** is templated from `.Values.domain.voicestudio` (both `http://`+`https://` forms). Behind the Olares proxy `origin_allowed` can compute the destination as `http` while the browser presents `https` → WS `/ws/events` handshakes rejected (1008 close, uvicorn logs them as `403 "connection rejected"`). Selling from the entrance's own domain fixes the CSRF/origin gate for every install of this chart, not just ours.
5. **`options.apiTimeout: 0`** — dubbing/audiobook/TTS generation runs for minutes; the entrance proxy default-caps requests at 15 s.
6. **`enableServiceLinks: false`** — the app's own `OMNIVOICE_*` env prefix can be clobbered by k8s legacy `*_PORT=tcp://...` injections when a same-namespace service shares the prefix.
7. **`OMNIVOICE_SERVER_MODE=1`** (baked in the image, pinned in env for clarity) relaxes the desktop-only loopback gate so the UI works behind the entrance proxy; admin mutations still require the API key. Do **not** set it to 0 here.
8. **`applications.app.bytetrade.io/gpu-inject: "true"`** on the Deployment is how app-service injects the `nvidia.com/gpumem` resources from the `spec.accelerator` envelope into the pod (verified against `facefusionv3`/`ollamav2` patterns).
9. **Model caching is `appCommon`** (`HF_HOME={{ .Values.userspace.appCommon }}/huggingface`, mount path == injected host path — misaligning mountPath vs value produced the first HF write-permission incident). `init-permissions` (beclab busybox, uid 0 — the only allowed root image) non-recursively chowns the mount roots; never `chown -R`.
10. **HF vars ride Olares' user-level AI settings** (`OLARES_USER_HUGGINGFACE_SERVICE/TOKEN` via `valueFrom`) so China-region installs get the configured mirror automatically. Template guards skip empty strings (Helm renders missing optional envs as `""` which apps choke on).

## 4. Upgrade playbook (upstream 0.5.x → newer)

**Step 0 — decide whether the port needs changes at all.** Read the target release's notes and scan `docs/install/docker.md` at that tag for:
- new/renamed `OMNIVOICE_*` env vars (server mode, bind host, worker tokens, ASR backends…)
- image tag changes, new ROCm variants, registry moves
- port or health-endpoint changes (`/health` shape), probe-relevant behavior

**Step 1 — verify the image exists.**
```sh
# Docker Hub digest for the new tag (expect 200; != none means OK)
curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:palashdeb/omnivoice-studio:pull" |
  python3 -c "import sys,json;print(json.load(sys.stdin)['token'])" | xargs -I{} \
  curl -s -o /dev/null -D - -H "Authorization: Bearer {}" \
  "https://registry-1.docker.io/v2/palashdeb/omnivoice-studio/manifests/<TAG>" | grep -i HTTP
```
Release tags can lag the GitHub release (0.5.5 published 2 images late). If newer tags don't exist yet, pin the newest published tag and note the lag in the changelog.

**Step 2 — edit the chart.** Exactly three coordinated spots:
- `Chart.yaml`: bump `version` (1.0.1 → 1.0.2 …) and `appVersion: "<upstream>"`.
- `OlaresManifest.yaml`: `metadata.version` == Chart `version` (lint enforces); `spec.versionName: "<upstream>"`.
- Both image references in `templates/deployment-voicestudio.yaml` (main container + `init-cudnn8`) — keep them in lockstep; the init container must match the main image's Python/pip.
Add new envs it introduces; drop removed ones. Re-check §3 quirks still apply if the image changed its entrypoint.

**Step 3 — validate, ship, verify.**
```sh
olares-cli chart lint ./charts/voicestudio
olares-cli chart package ./charts/voicestudio -o dist      # -> voicestudio-<new>.tgz
olares-cli market upload dist/voicestudio-<new>.tgz
# app already deployed -> upgrade (never install):
olares-cli market upgrade voicestudio -s upload --version <new> --watch --watch-timeout 1m -o json
```
Then verify at least the following (full evidence, not market state alone):
```sh
olares-cli cluster pod list -n voicestudio-my            # 2/2 Running, 0 restarts
olares-cli cluster container logs voicestudio-my/<pod>/voicestudio | tail   # startup lines
olares-cli settings apps list                            # URL column alive
```
Log greps that must be present/clean:
- `GPU pool sized to ...` — HAMi slice real
- `torch.compile applied` — identity envs working (not `"torch.compile deleted ... getpwuid"`)
- no `ERROR [core.cudnn8]` — cuDNN side-load found
- `/health` 200s from `127.0.0.1` (exec probes) **and** through the entrance

**Failure-recovery matrix**
| Symptom | Fix path |
|---|---|
| upgrade stuck / `upgradeFailed` | fix chart, bump, re-package, `market upgrade` again (never `install` over an existing app) |
| market row `stopped` but pods Running | `olares-cli market resume voicestudio --watch` |
| Upload `architecture_incompatible` | declared `supportArch` ∌ image arch — fix both or the chart version, NEVER bump-and-retry unchanged bytes |
| CrashLoop on first shape | `olares-cli cluster container logs ... --previous`; identity/probe/env quadrant in §3 |
| Entrée WS 403 | refresh app tab; if persistent, re-check `OMNIVOICE_ALLOWED_ORIGINS` env in the container |

## 5. Standing limitations & roadmap

- **Arch: `amd64` only.** Upstream publishes `linux/amd64` Docker images only (desktop runs macOS/ARM natively, but Docker ARM64 doesn't exist). Roadmap: build a **CPU-only `linux/arm64`** image from repo source in this repo's GitHub Actions (base `pytorch/pytorch:*-arm64`-compatible CPU image, no CUDA); until then `supportArch` stays `[amd64]` — declaring arm64 without an image is a false promise.
- **GPU slice is manifest-level.** The 6 Gi/16 Gi envelope lives in `spec.accelerator`; changing it = chart edit + version bump (deliberate control point). User still picks `cpu` vs `nvidia` at install time (`--compute-mode`).
- **Worker profiles ported-off.** The upstream compose's worker-gpu/worker-rocm profiles are intentionally not in this chart (headless GPU-lending to another machine is out of scope; add later as a separate chart if needed).
- **ROCm variant** untested on Olares AMD nodes — chart only declares `cpu`/`nvidia`.
- **Resources envelope** (2 CPU / 6–32 Gi RAM) is matched to a 24 GB+ node; consider trimming `limitedMemory` for 16 GB boxes in a future patch.

## 6. Distribution housekeeping (this repo)

- Chart source of truth: `charts/voicestudio/`. `dist/` is build output (gitignored).
- Tags: **`chart-v<w+x+y>` must equal Chart.yaml `version`.** The release workflow packages + publishes the `.tgz` on that tag.
- Installers pull the latest release asset — so bump + tag for every chart change, even metadata-only.
- Image registry stays upstream (`palashdeb/omnivoice-studio`); only if we stop depending on upstream's CI should this repo gain its own image workflows.