# VoiceStudio on Olares

Community [Olares](https://www.olares.com) chart for [VoiceStudio](https://github.com/debpalash/VoiceStudio) — the open-source, fully-local **ElevenLabs alternative**: voice cloning, voice design, video dubbing, dictation, transcription, and audiobook creation in 646 languages.

This chart ships VoiceStudio's **headless web-server image** as an Olares app: your audio, voices, and projects stay on your machine; models download into Olares' cross-app shared model cache; an NVIDIA GPU slice is used when available.

![VoiceStudio Model Catalogue](https://raw.githubusercontent.com/debpalash/VoiceStudio/main/docs/media/0.5.0/catalogue.png)

## Requirements

| | |
|---|---|
| Olares | 1.12.6+ (`olares-cli` logged in) |
| Arch | amd64 (upstream Docker images are amd64-only; arm64 CPU build is on the roadmap — see `PLAN-voicestudio.md`) |
| NVIDIA (optional) | driver + 8 GB+ VRAM recommended; the app gets a bounded 6 Gi floor / 16 Gi cap vGPU slice |
| CPU-only | 8 GB+ free RAM; low-footprint engine family (PocketTTS / Sherpa-ONNX / KittenTTS, Faster-Whisper int8) |
| Disk | ~10 GB floor for data; model weights land in the shared cache (up to tens of GB as you install more engines) |

## Install

### From a release (recommended)

1. Download the latest `voicestudio-<version>.tgz` from this repo's [Releases](https://github.com/abidals/VoiceStudio-Olares/releases).
2. Generate an admin API key and install:

```sh
API_KEY="$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')"
echo "$API_KEY"   # save this — the app will ask for it

olares-cli market upload ./voicestudio-<version>.tgz
olares-cli market install voicestudio -s upload --version <version> \
  --compute-mode nvidia \
  --env OMNIVOICE_API_KEY="$API_KEY"
```

Use `--compute-mode cpu` on machines without an NVIDIA GPU.

### From source

```sh
git clone https://github.com/abidals/VoiceStudio-Olares.git
cd VoiceStudio-Olares
olares-cli chart lint ./charts/voicestudio
olares-cli chart package ./charts/voicestudio -o dist
# then upload + install as above
```

## First run

1. Open the app from the Olares Desktop (the URL is also on the install card).
2. The UI prompts once for your `OMNIVOICE_API_KEY`; it then holds a short-lived session.
3. The default **OmniVoice** model (~2.4 GB) downloads **on first use** into the shared cache — watch progress in the app or via `olares-cli cluster container logs voicestudio-<you>/<pod>/voicestudio`. Every engine downloads its own weights the first time you select it.
4. WhisperX / Faster-Whisper work out of the box: the chart side-loads cuDNN 8 into the app data on first boot.

## Sync with native apps

Point the native [VoiceStudio desktop app](https://github.com/debpalash/VoiceStudio/releases) (or any OpenAI-compatible client) at your instance:

- Backend URL: `https://<your-entrance-host>`
- API key: your `OMNIVOICE_API_KEY`
- OpenAI-compatible audio API: `https://<your-entrance-host>/v1` (`/v1/audio/speech`, `/v1/audio/transcriptions`, MCP at `/mcp`)

The entrance is reachable without an Olares login so external apps can sync — the app itself requires the API key for every rule (downloads, settings, generation), so keep the key secret.

## Local AI gateway

If your Olares has a local AI gateway configured (Settings → AI), copy its **base URL + key** into VoiceStudio's **Model Catalogue → OpenAI-compatible ASR engine** to transcribe through it. The chart also auto-injects your Olares-managed Hugging Face endpoint/token for model downloads.

## GPU slice

In `nvidia` mode the app reserves a **bounded VRAM slice** (6 Gi floor, 16 Gi cap by default) via Olares' HAMi scheduler; the rest of the card stays available to other apps. Inference scales to what was reserved. To change the slice, edit `spec.accelerator` in `charts/voicestudio/OlaresManifest.yaml`, bump the chart version, and upgrade — details in [`PLAN-voicestudio.md`](PLAN-voicestudio.md).

## Uninstall

```sh
olares-cli market uninstall voicestudio --watch
```
Your data remains in `AppData/voicestudio` (delete it from Files if you want a full wipe).

## Troubleshooting

| Symptom | See |
|---|---|
| Deep-dive: port quirks, uid-1000 identity, probe design, cuDNN side-load, upgrade playbook | [`PLAN-voicestudio.md`](PLAN-voicestudio.md) |
| App-side logs, self-check, engine docs | [Upstream troubleshooting](https://github.com/debpalash/VoiceStudio/blob/main/docs/install/troubleshooting.md) |

## License

Chart and workflow: AGPL-3.0, same as [upstream VoiceStudio](https://github.com/debpalash/VoiceStudio/blob/main/LICENSE). Downloaded models keep their upstream licenses (OmniVoice weights: CC-BY-NC).

Maintainer: [@abidals](https://github.com/abidals)