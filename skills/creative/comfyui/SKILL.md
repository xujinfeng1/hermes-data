---
name: comfyui
description: "Generate images, video, and audio with ComfyUI — install, launch, manage nodes/models, run workflows with parameter injection. Uses the official comfy-cli for lifecycle and direct REST/WebSocket API for execution."
version: 5.1.0
author: [kshitijk4poor, alt-glitch, purzbeats]
license: MIT
platforms: [macos, linux, windows]
compatibility: "Requires ComfyUI (local, Comfy Desktop, or Comfy Cloud) and comfy-cli (auto-installed via pipx/uvx by the setup script)."
prerequisites:
  commands: ["python3"]
setup:
  help: "Run scripts/hardware_check.py FIRST to decide local vs Comfy Cloud; then scripts/comfyui_setup.sh auto-installs locally (or use Cloud API key for platform.comfy.org)."
metadata:
  hermes:
    tags:
      - comfyui
      - image-generation
      - stable-diffusion
      - flux
      - sd3
      - wan-video
      - hunyuan-video
      - creative
      - generative-ai
      - video-generation
    related_skills: [stable-diffusion-image-generation, image_gen]
    category: creative
---

# ComfyUI

Generate images, video, audio, and 3D content through ComfyUI using the
official `comfy-cli` for setup/lifecycle and direct REST/WebSocket API
for workflow execution.

## What's in this skill

**Reference docs (`references/`):**

- `official-cli.md` — every `comfy ...` command, with flags
- `rest-api.md` — REST + WebSocket endpoints (local + cloud), payload schemas
- `workflow-format.md` — API-format JSON, common node types, param mapping
- `template-integrity.md` — converting `comfyui-workflow-templates` from editor format to API format
- `aigc-studio-wrapper.md` — FastAPI web wrapper pattern, job queue, Chinese prompt handling, UI patterns
- `china-setup.md` — 中国大陆安装指南：HF 镜像、Apple Silicon 验证、网络问题排查
- `references/template-integrity.md` — converting `comfyui-workflow-templates` from
  editor format to API format: Reroute bypass, dotted dynamic-input keys
- `aigc-studio-rules.md` — **CRITICAL: development rules for ComfyUI wrappers**:
  diagnose-before-code, translation-before-apply_params, JS validation, M1 Max limits
- `translation-order.md` — critical pitfall: translating Chinese prompts BEFORE
  `apply_params`, not after. Two-layer backstop pattern (endpoint + engine).
- `js-validation.md` — inline JS validation pattern using `node -e "new Function()"`,
  common corruptions (orphaned async, over-escaped quotes, duplicate functions)
  (`values.a`, `resize_type.width`), Cloud quirks (302 redirect, 1 concurrent
  free-tier job, 1080p VRAM ceiling), Discord-compatible ffmpeg stitch.
  Authored by [@purzbeats](https://github.com/purzbeats). Load this whenever
  you're starting from an official template.

**Scripts (`scripts/`):**

| Script | Purpose |
|--------|---------|
| `_common.py` | Shared HTTP, cloud routing, node catalogs (don't run directly) |
| `hardware_check.py` | Probe GPU/VRAM/disk → recommend local vs Comfy Cloud |
| `comfyui_setup.sh` | Hardware check + comfy-cli + ComfyUI install + launch + verify |
| `extract_schema.py` | Read a workflow → list controllable params + model deps |
| `check_deps.py` | Check workflow against running server → list missing nodes/models |
| `auto_fix_deps.py` | Run check_deps then `comfy node install` / `comfy model download` |
| `run_workflow.py` | Inject params, submit, monitor, download outputs (HTTP or WS) |
| `run_batch.py` | Submit a workflow N times with sweeps, parallel up to your tier |
| `ws_monitor.py` | Real-time WebSocket viewer for executing jobs (live progress) |
| `health_check.py` | Verification checklist runner — comfy-cli + server + models + smoke test |
| `fetch_logs.py` | Pull traceback / status messages for a given prompt_id |

**Example workflows (`workflows/`):** SD 1.5, SDXL, Flux Dev, SDXL img2img,
SDXL inpaint, ESRGAN upscale, AnimateDiff video, Wan T2V. See
`workflows/README.md`.

- `animatediff-workflows.md` — AnimateDiff SD1.5 video workflows
- `aigc-studio-pattern.md` — Web wrapper architecture, async job queue, prompt handling


## When to Use

- User asks to generate images with Stable Diffusion, SDXL, Flux, SD3, etc.
- User wants to run a specific ComfyUI workflow file
- User wants to chain generative steps (txt2img → upscale → face restore)
- User needs ControlNet, inpainting, img2img, or other advanced pipelines
- User asks to manage ComfyUI queue, check models, or install custom nodes
- User wants video/audio/3D generation via AnimateDiff, Hunyuan, Wan, AudioCraft, etc.

## Architecture: Two Layers

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: comfy-cli (official lifecycle tool)        │
│   Setup, server lifecycle, custom nodes, models     │
│   → comfy install / launch / stop / node / model    │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│ Layer 2: REST/WebSocket API + skill scripts         │
│   Workflow execution, param injection, monitoring   │
│   POST /api/prompt, GET /api/view, WS /ws           │
│   → run_workflow.py, run_batch.py, ws_monitor.py    │
└─────────────────────────────────────────────────────┘
```

**Why two layers?** The official CLI is excellent for installation and server
management but has minimal workflow execution support. The REST/WS API fills
that gap — the scripts handle param injection, execution monitoring, and
output download that the CLI doesn't do.

## Quick Start

### Detect environment

```bash
# What's available?
command -v comfy >/dev/null 2>&1 && echo "comfy-cli: installed"
curl -s http://127.0.0.1:8188/system_stats 2>/dev/null && echo "server: running"

# Can this machine run ComfyUI locally? (GPU/VRAM/disk check)
python3 scripts/hardware_check.py
```

If nothing is installed, see **Setup & Onboarding** below — but always run the
hardware check first.

### One-line health check

```bash
python3 scripts/health_check.py
# → JSON: comfy_cli on PATH? server reachable? at least one checkpoint? smoke-test passes?
```

## Core Workflow

### Step 1: Get a workflow JSON in API format

Workflows must be in API format (each node has `class_type`). They come from:

- ComfyUI web UI → **Workflow → Export (API)** (newer UI) or
  the legacy "Save (API Format)" button (older UI)
- This skill's `workflows/` directory (ready-to-run examples)
- Community downloads (civitai, Reddit, Discord) — usually editor format,
  must be loaded into ComfyUI then re-exported

Editor format (top-level `nodes` and `links` arrays) is **not directly
executable**. The scripts detect this and tell you to re-export.

### Step 2: See what's controllable

```bash
python3 scripts/extract_schema.py workflow_api.json --summary-only
# → {"parameter_count": 12, "has_negative_prompt": true, "has_seed": true, ...}

python3 scripts/extract_schema.py workflow_api.json
# → full schema with parameters, model deps, embedding refs
```

### Step 3: Run with parameters

```bash
# Local (defaults to http://127.0.0.1:8188)
python3 scripts/run_workflow.py \
  --workflow workflow_api.json \
  --args '{"prompt": "a beautiful sunset over mountains", "seed": -1, "steps": 30}' \
  --output-dir ./outputs

# Cloud (export API key once; uses correct /api routing automatically)
export COMFY_CLOUD_API_KEY="comfyui-..."
python3 scripts/run_workflow.py \
  --workflow workflow_api.json \
  --args '{"prompt": "..."}' \
  --host https://cloud.comfy.org \
  --output-dir ./outputs

# Real-time progress via WebSocket (requires `pip install websocket-client`)
python3 scripts/run_workflow.py \
  --workflow flux_dev.json \
  --args '{"prompt": "..."}' \
  --ws

# img2img / inpaint: pass --input-image to upload + reference automatically
python3 scripts/run_workflow.py \
  --workflow sdxl_img2img.json \
  --input-image image=./photo.png \
  --args '{"prompt": "make it watercolor", "denoise": 0.6}'

# Batch / sweep: 8 random seeds, parallel up to cloud tier limit
python3 scripts/run_batch.py \
  --workflow sdxl.json \
  --args '{"prompt": "abstract"}' \
  --count 8 --randomize-seed --parallel 3 \
  --output-dir ./outputs/batch
```

`-1` for `seed` (or omitting it with `--randomize-seed`) generates a fresh
random seed per run.

### Step 4: Present results

The scripts emit JSON to stdout describing every output file:

```json
{
  "status": "success",
  "prompt_id": "abc-123",
  "outputs": [
    {"file": "./outputs/sdxl_00001_.png", "node_id": "9",
     "type": "image", "filename": "sdxl_00001_.png"}
  ]
}
```

## Decision Tree

| User says | Tool | Command |
|-----------|------|---------|
| **Lifecycle (use comfy-cli)** | | |
| "install ComfyUI" | comfy-cli | `bash scripts/comfyui_setup.sh` |
| "start ComfyUI" | comfy-cli | `comfy launch --background` |
| "stop ComfyUI" | comfy-cli | `comfy stop` |
| "install X node" | comfy-cli | `comfy node install <name>` |
| "download X model" | comfy-cli | `comfy model download --url <url> --relative-path models/checkpoints` |
| "list installed models" | comfy-cli | `comfy model list` |
| "list installed nodes" | comfy-cli | `comfy node show installed` |
| **Execution (use scripts)** | | |
| "is everything ready?" | script | `health_check.py` (optionally with `--workflow X --smoke-test`) |
| "what can I change in this workflow?" | script | `extract_schema.py W.json` |
| "check if W's deps are met" | script | `check_deps.py W.json` |
| "fix missing deps" | script | `auto_fix_deps.py W.json` |
| "generate an image" | script | `run_workflow.py --workflow W --args '{...}'` |
| "use this image" (img2img) | script | `run_workflow.py --input-image image=./x.png ...` |
| "8 variations with random seeds" | script | `run_batch.py --count 8 --randomize-seed ...` |
| "show me live progress" | script | `ws_monitor.py --prompt-id <id>` |
| "fetch the error from job X" | script | `fetch_logs.py <prompt_id>` |
| **Direct REST** | | |
| "what's in the queue?" | REST | `curl http://HOST:8188/queue` (local) or `--host https://cloud.comfy.org` |
| "cancel that" | REST | `curl -X POST http://HOST:8188/interrupt` |
| "free GPU memory" | REST | `curl -X POST http://HOST:8188/free` |

## Setup & Onboarding

When a user asks to set up ComfyUI, **the FIRST thing to do is ask whether
they want Comfy Cloud (hosted, zero install, API key) or Local (install
ComfyUI on their machine)**. Don't start running install commands or hardware
checks until they've answered.

**Official docs:** https://docs.comfy.org/installation
**CLI docs:** https://docs.comfy.org/comfy-cli/getting-started
**Cloud docs:** https://docs.comfy.org/get_started/cloud
**Cloud API:** https://docs.comfy.org/development/cloud/overview

### Step 0: Ask Local vs Cloud (ALWAYS FIRST)

Suggested script:

> "Do you want to run ComfyUI locally on your machine, or use Comfy Cloud?
>
> - **Comfy Cloud** — hosted on RTX 6000 Pro GPUs, all common models pre-installed,
>   zero setup. Requires an API key (paid subscription required to actually run
>   workflows; free tier is read-only). Best if you don't have a capable GPU.
> - **Local** — free, but your machine MUST meet the hardware requirements:
>   - NVIDIA GPU with **≥6 GB VRAM** (≥8 GB for SDXL, ≥12 GB for Flux/video), OR
>   - AMD GPU with ROCm support (Linux), OR
>   - Apple Silicon Mac (M1+) with **≥16 GB unified memory** (≥32 GB recommended).
>   - Intel Macs and machines with no GPU will NOT work — use Cloud instead.
>
> Which would you like?"

Routing:

- **Cloud** → skip to **Path A**.
- **Local** → run hardware check first, then pick a path from Paths B–E based on the verdict.
- **Unsure** → run the hardware check and let the verdict decide.

### Step 1: Verify Hardware (ONLY if user chose local)

```bash
python3 scripts/hardware_check.py --json
# Optional: also probe `torch` for actual CUDA/MPS:
python3 scripts/hardware_check.py --json --check-pytorch
```

| Verdict    | Meaning                                                       | Action |
|------------|---------------------------------------------------------------|--------|
| `ok`       | ≥8 GB VRAM (discrete) OR ≥32 GB unified (Apple Silicon)       | Local install — use `comfy_cli_flag` from report |
| `marginal` | SD1.5 works; SDXL tight; Flux/video unlikely                  | Local OK for light workflows, else **Path A (Cloud)** |
| `cloud`    | No usable GPU, <6 GB VRAM, <16 GB Apple unified, Intel Mac, Rosetta Python | **Switch to Cloud** unless user explicitly forces local |

The script also surfaces `wsl: true` (WSL2 with NVIDIA passthrough) and
`rosetta: true` (x86_64 Python on Apple Silicon — must reinstall as ARM64).

If verdict is `cloud` but the user wants local, do not proceed silently.
Show the `notes` array verbatim and ask whether they want to (a) switch to
Cloud or (b) force a local install (will OOM or be unusably slow on modern models).

### Choosing an Installation Path

Use the hardware check first. The table below is the fallback for when the
user has already told you their hardware:

| Situation | Recommended Path |
|-----------|------------------|
| `verdict: cloud` from hardware check | **Path A: Comfy Cloud** |
| No GPU / want to try without commitment | **Path A: Comfy Cloud** |
| Windows + NVIDIA + non-technical | **Path B: ComfyUI Desktop** |
| Windows + NVIDIA + technical | **Path C: Portable** or **Path D: comfy-cli** |
| Linux + any GPU | **Path D: comfy-cli** (easiest) |
| macOS + Apple Silicon | **Path B: Desktop** or **Path D: comfy-cli** |
| Headless / server / CI / agents | **Path D: comfy-cli** |

For the fully automated path (hardware check → install → launch → verify):

```bash
bash scripts/comfyui_setup.sh
# Or with overrides:
bash scripts/comfyui_setup.sh --m-series --port=8190 --workspace=/data/comfy
```

It runs `hardware_check.py` internally, refuses to install locally when the
verdict is `cloud` (unless `--force-cloud-override`), picks the right
`comfy-cli` flag, and prefers `pipx`/`uvx` over global `pip` to avoid polluting
system Python.

---

### Path A: Comfy Cloud (No Local Install)

For users without a capable GPU or who want zero setup. Hosted on RTX 6000 Pro.

**Docs:** https://docs.comfy.org/get_started/cloud

1. Sign up at https://comfy.org/cloud
2. Generate an API key at https://platform.comfy.org/login
3. Set the key:
   ```bash
   export COMFY_CLOUD_API_KEY="comfyui-xxxxxxxxxxxx"
   ```
4. Run workflows:
   ```bash
   python3 scripts/run_workflow.py \
     --workflow workflows/flux_dev_txt2img.json \
     --args '{"prompt": "..."}' \
     --host https://cloud.comfy.org \
     --output-dir ./outputs
   ```

**Pricing:** https://www.comfy.org/cloud/pricing
**Concurrent jobs:** Free/Standard 1, Creator 3, Pro 5. Free tier
**cannot run workflows via API** — only browse models. Paid subscription
required for `/api/prompt`, `/api/upload/*`, `/api/view`, etc.

---

### Path B: ComfyUI Desktop (Windows / macOS)

One-click installer for non-technical users. Currently Beta.

**Docs:** https://docs.comfy.org/installation/desktop
- **Windows (NVIDIA):** https://download.comfy.org/windows/nsis/x64
- **macOS (Apple Silicon):** https://comfy.org

Linux is **not supported** for Desktop — use Path D.

---

### Path C: ComfyUI Portable (Windows Only)

**Docs:** https://docs.comfy.org/installation/comfyui_portable_windows

Download from https://github.com/comfyanonymous/ComfyUI/releases, extract,
run `run_nvidia_gpu.bat`. Update via `update/update_comfyui_stable.bat`.

---

### Path D: comfy-cli (All Platforms — Recommended for Agents)

The official CLI is the best path for headless/automated setups.

**Docs:** https://docs.comfy.org/comfy-cli/getting-started

#### Install comfy-cli

```bash
# Recommended:
pipx install comfy-cli
# Or use uvx without installing:
uvx --from comfy-cli comfy --help
# Or (if pipx/uvx unavailable):
pip install --user comfy-cli
```

Disable analytics non-interactively:
```bash
comfy --skip-prompt tracking disable
```

#### Install ComfyUI

```bash
comfy --skip-prompt install --nvidia              # NVIDIA (CUDA)
comfy --skip-prompt install --amd                 # AMD (ROCm, Linux)
comfy --skip-prompt install --m-series            # Apple Silicon (MPS)
comfy --skip-prompt install --cpu                 # CPU only (slow)
comfy --skip-prompt install --nvidia --fast-deps  # uv-based dep resolution
```

Default location: `~/comfy/ComfyUI` (Linux), `~/Documents/comfy/ComfyUI`
(macOS/Win). Override with `comfy --workspace /custom/path install`.

#### Launch / verify

```bash
comfy launch --background                       # background daemon on :8188
comfy launch -- --listen 0.0.0.0 --port 8190    # LAN-accessible custom port
curl -s http://127.0.0.1:8188/system_stats      # health check
```

---

### Path E: Manual Install (Advanced / Unsupported Hardware)

For Ascend NPU, Cambricon MLU, Intel Arc, or other unsupported hardware.

**Docs:** https://docs.comfy.org/installation/manual_install

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu130
pip install -r requirements.txt
python main.py
```

---

### Post-Install: Download Models

```bash
# SDXL (general purpose, ~6.5 GB)
comfy model download \
  --url "https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors" \
  --relative-path models/checkpoints

# SD 1.5 (lighter, ~4 GB, good for 6 GB cards)
comfy model download \
  --url "https://huggingface.co/stable-diffusion-v1-5/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors" \
  --relative-path models/checkpoints

# Flux Dev fp8 (smaller variant, ~12 GB)
comfy model download \
  --url "https://huggingface.co/Comfy-Org/flux1-dev/resolve/main/flux1-dev-fp8.safetensors" \
  --relative-path models/checkpoints

# CivitAI (set token first):
comfy model download \
  --url "https://civitai.com/api/download/models/128713" \
  --relative-path models/checkpoints \
  --set-civitai-api-token "YOUR_TOKEN"
```

List installed: `comfy model list`.

### Post-Install: Install Custom Nodes

```bash
comfy node install comfyui-impact-pack             # popular utility pack
comfy node install comfyui-animatediff-evolved     # video generation
comfy node install comfyui-controlnet-aux          # ControlNet preprocessors
comfy node install comfyui-essentials              # common helpers
comfy node update all
comfy node install-deps --workflow=workflow.json   # install everything a workflow needs
```

### Post-Install: Verify

```bash
python3 scripts/health_check.py
# → comfy_cli on PATH? server reachable? checkpoints? smoke test?

python3 scripts/check_deps.py my_workflow.json
# → are this workflow's nodes/models/embeddings installed?

python3 scripts/run_workflow.py \
  --workflow workflows/sd15_txt2img.json \
  --args '{"prompt": "test", "steps": 4}' \
  --output-dir ./test-outputs
```

## Image Upload (img2img / Inpainting)

The simplest way is to use `--input-image` with `run_workflow.py`:

```bash
python3 scripts/run_workflow.py \
  --workflow workflows/sdxl_img2img.json \
  --input-image image=./photo.png \
  --args '{"prompt": "make it cyberpunk", "denoise": 0.6}'
```

The flag uploads `photo.png`, then injects its server-side filename into
whatever schema parameter is named `image`. For inpainting, pass both:

```bash
python3 scripts/run_workflow.py \
  --workflow workflows/sdxl_inpaint.json \
  --input-image image=./photo.png \
  --input-image mask_image=./mask.png \
  --args '{"prompt": "fill with flowers"}'
```

Manual upload via REST:
```bash
curl -X POST "http://127.0.0.1:8188/upload/image" \
  -F "image=@photo.png" -F "type=input" -F "overwrite=true"
# Returns: {"name": "photo.png", "subfolder": "", "type": "input"}

# Cloud equivalent:
curl -X POST "https://cloud.comfy.org/api/upload/image" \
  -H "X-API-Key: $COMFY_CLOUD_API_KEY" \
  -F "image=@photo.png" -F "type=input" -F "overwrite=true"
```

## Cloud Specifics

- **Base URL:** `https://cloud.comfy.org`
- **Auth:** `X-API-Key` header (or `?token=KEY` for WebSocket)
- **API key:** set `$COMFY_CLOUD_API_KEY` once and the scripts pick it up automatically
- **Output download:** `/api/view` returns a 302 to a signed URL; the scripts
  follow it and strip `X-API-Key` before fetching from the storage backend
  (don't leak the API key to S3/CloudFront).
- **Endpoint differences from local ComfyUI:**
  - `/api/object_info`, `/api/queue`, `/api/userdata` — **403 on free tier**;
    paid only.
  - `/history` is renamed to `/history_v2` on cloud (the scripts route
    automatically).
  - `/models/<folder>` is renamed to `/experiment/models/<folder>` on cloud
    (the scripts route automatically).
  - `clientId` in WebSocket is currently ignored — all connections for a
    user receive the same broadcast. Filter by `prompt_id` client-side.
  - `subfolder` is accepted on uploads but ignored — cloud has a flat namespace.
- **Concurrent jobs:** Free/Standard: 1, Creator: 3, Pro: 5. Extras queue
  automatically. Use `run_batch.py --parallel N` to saturate your tier.

## Queue & System Management

```bash
# Local
curl -s http://127.0.0.1:8188/queue | python3 -m json.tool
curl -X POST http://127.0.0.1:8188/queue -d '{"clear": true}'    # cancel pending
curl -X POST http://127.0.0.1:8188/interrupt                      # cancel running
curl -X POST http://127.0.0.1:8188/free \
  -H "Content-Type: application/json" \
  -d '{"unload_models": true, "free_memory": true}'

# Cloud — same paths under /api/, plus:
python3 scripts/fetch_logs.py --tail-queue --host https://cloud.comfy.org
```

## SDXL-Specific Pitfalls

1. **CLIPTextEncodeSDXL required** — SDXL uses dual text encoders (CLIP-L + CLIP-G).
   Using plain `CLIPTextEncode` causes prompts to be silently ignored — images won't match
   the prompt at all. Always use `CLIPTextEncodeSDXL` node with both `text_g` and `text_l`
   fields set to the same prompt/negative prompt.

2. **Chinese prompts don't work** — SDXL is trained on English. Chinese text produces
   noise. Always auto-translate via LLM before sending to ComfyUI:
   ```python
   if any('\u4e00' <= c <= '\u9fff' for c in prompt):
       result = await llm_optimize(prompt)  # translate to English
   ```

3. **Img2img image upload** — Must upload input image to ComfyUI's `/upload/image`
   endpoint FIRST, get the returned filename, then inject that filename into the
   `LoadImage` node's `image` field. The Web UI's FileReader gives base64 — convert
   to bytes, POST to ComfyUI, use the returned `{"name": "file.png"}`.

4. **Denoise for img2img** — Default `0.75` is too high for content preservation.
   Industry ranges: 0.2-0.3 (filter only), 0.4-0.5 (style change, preserve composition),
   0.7-0.9 (major change, like txt2img).

5. **Output format parsing** — ComfyUI history returns nested dict:
   `history["outputs"] = {node_id: {"images": [{"filename": "...", "subfolder": "", "type": "output"}]}}`.
   Parse `images` from each node, not the node output directly.

6. **Video without custom nodes** — Generate N frames with different seeds, download
   each, then stitch into GIF with PIL or MP4 with ffmpeg. No AnimateDiff/Wan nodes
   needed. 8 frames × ~30s each ≈ 4 minutes total.

7. **Mac MPS-specific** — First SDXL run on Apple Silicon takes extra time to warm
   the model. Subsequent runs are cached and faster. Use `--m-series` flag for
   comfy-cli install.

8. **China network** — HuggingFace downloads are blocked. Use hf-mirror.com:
   ```bash
   curl -L -o model.safetensors "https://hf-mirror.com/USER/REPO/resolve/main/FILE"
   ```

## Pitfalls

1. **API format required** — every script and the `/api/prompt` endpoint expect
   API-format workflow JSON...

2. **SDXL 必须用 CLIPTextEncodeSDXL** — 详见 `references/china-pitfalls.md`
   中文提示词 = 随机噪声，必须 LLM 翻译后再提交。

3. **AnimateDiff 工作流验证** — 节点输入/输出类型可用 `/object_info/<NodeName>` API 查询，
   直接提交验证: `POST /prompt` 返回 `node_errors` 精确指出问题。

4. **终端不走 VPN** — curl/python 请求不走系统 VPN。
   需要 VPN 下载的模型让用户浏览器下载后手动放置。

10. **Auto-randomized seed**

0. **SDXL 不吃中文** — SDXL 训练数据全是英文，中文 prompt 等于随机噪声。必须在发给 ComfyUI 之前用 LLM 把中文翻译成英文。检测方法：`any('\u4e00' <= c <= '\u9fff' for c in prompt)`。

0. **ComfyUI 长时间运行会卡死** — MPS/GPU 内存碎片化导致 HTTP 无响应。症状：进程在但 `/system_stats` 超时。解决：`kill` 后重新 `comfy launch --background`。

0. **网络环境（中国大陆）** — HuggingFace 直连被墙，hf-mirror.com 不稳定。ModelScope (modelscope.cn) 对 SD1.5/SDXL 等常见模型可用。运动模块（AnimateDiff）所有国内源均不可用，需 VPN 浏览器下载。

0. **终端不走系统 VPN** — macOS 上即使开启了 VPN，终端 `curl` 等命令不会自动走代理。需设置 `https_proxy` 环境变量或用浏览器下载后手动移文件。

1. **API format required** — every script and the `/api/prompt` endpoint expect
   API-format workflow JSON. The scripts detect editor format (top-level
   `nodes` and `links` arrays) and tell you to re-export via
   "Workflow → Export (API)" (newer UI) or "Save (API Format)" (older UI).

2. **Server must be running** — all execution requires a live server.
   `comfy launch --background` starts one. Verify with
   `curl http://127.0.0.1:8188/system_stats`.

3. **Model names are exact** — case-sensitive, includes file extension.
   `check_deps.py` does fuzzy matching (with/without extension and folder
   prefix), but the workflow itself must use the canonical name. Use
   `comfy model list` to discover what's installed.

4. **Missing custom nodes** — "class_type not found" means a required node
   isn't installed. `check_deps.py` reports which package to install;
   `auto_fix_deps.py` runs the install for you.

5. **Working directory** — `comfy-cli` auto-detects the ComfyUI workspace.
   If commands fail with "no workspace found", use
   `comfy --workspace /path/to/ComfyUI <command>` or
   `comfy set-default /path/to/ComfyUI`.

6. **Cloud free-tier API limits** — `/api/prompt`, `/api/view`, `/api/upload/*`,
   `/api/object_info` all return 403 on free accounts. `health_check.py` and
   `check_deps.py` handle this gracefully and surface a clear message.

7. **Timeout for video/audio workflows** — auto-detected when an output node
   is `VHS_VideoCombine`, `SaveVideo`, etc.; the default jumps from 300 s to
   900 s. Override explicitly with `--timeout 1800`.

8. **Path traversal in output filenames** — server-supplied filenames are
   passed through `safe_path_join` to refuse anything escaping `--output-dir`.
   Keep this protection on — workflows with custom save nodes can produce
   arbitrary paths.

9. **Workflow JSON is arbitrary code** — custom nodes run Python, so
   submitting an unknown workflow has the same trust profile as `eval`.
   Inspect workflows from untrusted sources before running.

10. **SDXL requires CLIPTextEncodeSDXL, not CLIPTextEncode** — SDXL uses
    dual text encoders (CLIP-L + CLIP-G). Using plain `CLIPTextEncode` on
    an SDXL workflow silently drops one encoder, causing the model to
    barely follow the prompt (images appear unrelated to text). Always use
    `CLIPTextEncodeSDXL` with both `text_g` and `text_l` inputs when the
    checkpoint is SDXL. If the user reports "画面与提示词完全无关" on SDXL,
    this is the first thing to check.

11. **Chinese prompt → SDXL = random noise** — SDXL training data is
    English-only. Chinese text produces garbage. Auto-translate Chinese
    prompts to English before injecting into the workflow. Detection:
    `any('\u4e00' <= c <= '\u9fff' for c in prompt)`.

12. **M1/M2/M3 Mac ComfyUI stability** — After extended runtime (hours),
    ComfyUI on Apple Silicon may hang (MPS memory fragmentation).
    Symptoms: HTTP server stops responding but process is still alive and
    using memory. Fix: restart ComfyUI (`comfy launch --background` after
    killing the old process). For production use on Mac, schedule periodic
    restarts or monitor `/system_stats` with a health-check loop.

13. **Model downloads from China** — Download priority order:
    1. `modelscope.cn` — SD1.5, SDXL commonly available, ~5MB/s
    2. `hf-mirror.com` — most HF repos mirrored, ~3MB/s but intermittent
    3. `ghproxy.net` — GitHub repos only, unreliable for HF
    4. Direct `huggingface.co` — blocked, do not attempt without VPN
    Less-common models (motion modules, LoRAs, custom checkpoints) often
    fail on all Chinese mirrors. Use VPN as fallback.

11. **Auto-randomized seed** — pass `seed: -1` in `--args` (or use
    `--randomize-seed` and omit the seed) to get a fresh seed per run.
    The actual seed is logged to stderr.

12. **SDXL workflow requires `CLIPTextEncodeSDXL`, not `CLIPTextEncode`** —
    SDXL has dual text encoders (CLIP-L + CLIP-G). Using plain
    `CLIPTextEncode` causes unrelated output. Always use
    `CLIPTextEncodeSDXL` with both `text_g` and `text_l` inputs when
    the checkpoint is SDXL-based. The `CheckpointLoaderSimple` output
    clip connection auto-supplies both encoders.

13. **SDXL does not understand Chinese** — all prompts must be in
    English. If building a Chinese-facing UI, auto-translate prompts
    via an LLM before submitting to ComfyUI. Chinese text produces
    noise/unrelated images.

14. **img2img requires uploading the reference image to ComfyUI FIRST**
    — POST `multipart/form-data` to `/upload/image` with `type=input`
    to get back a `{"name": "file.png"}`. Inject that filename into
    the `LoadImage` node's `image` field. The workflow cannot
    reference local filesystem paths.

15. **Denoise defaults** — for img2img, denoise 0.5 is a good default
    that preserves the original image while allowing style transfer.
    0.3-0.5 = heavy preservation, 0.5-0.7 = moderate change, 0.7-0.9
    = major transformation.

16. **ComfyUI output format** — history outputs are nested:
    `{"node_id": {"images": [{"filename": "..."}]}}`. When
    downloading, iterate `outputs[node_id]["images"]`, not
    `outputs[node_id]` directly. Some nodes return `{"gifs": [...]}`
    instead — handle both.

17. **Video generation without custom nodes** — generate N frames with
    different seeds, then stitch into GIF with PIL:
    ```python
    images[0].save("out.gif", save_all=True, append_images=images[1:],
                   duration=250, loop=0)
    ```
    This requires no model downloads beyond the base SDXL checkpoint.

11. **China network: HuggingFace downloads blocked** — direct downloads from
    huggingface.co time out or crawl in mainland China. Use the mirror
    (`hf-mirror.com`) for model downloads: replace `huggingface.co` with
    `hf-mirror.com` in the URL. Also set `HF_ENDPOINT=https://hf-mirror.com`
    for Python libraries. When `comfy model download` hangs silently, kill it
    and use `curl -L -o <output> "<mirror-url>"` directly.

12. **SDXL requires CLIPTextEncodeSDXL, not CLIPTextEncode** — SDXL uses dual
    text encoders (CLIP-L + CLIP-G). Using the single `CLIPTextEncode` node
    with an SDXL checkpoint causes COMPLETELY unrelated images because only
    CLIP-L is utilized. Always use `CLIPTextEncodeSDXL` with both `text_g`
    and `text_l` inputs. For API-submitted workflows, the node must have:
    ```json
    {"class_type":"CLIPTextEncodeSDXL","inputs":{"text_g":"PROMPT","text_l":"PROMPT","width":1024,"height":1024,"crop_w":0,"crop_h":0,"target_width":1024,"target_height":1024,"clip":["4",1]}}
    ```

13. **SDXL / Stable Diffusion CANNOT understand Chinese prompts** — All
    Stable-Diffusion-family models (SD1.5, SDXL, Flux) are trained on English
    text-image pairs. Chinese text is treated as random noise, producing
    completely unrelated images. Always auto-translate Chinese prompts to
    English before submission. Use an LLM (DeepSeek/GPT) to translate and
    expand into a descriptive English prompt with quality keywords. Auto-detect
    Chinese characters (`\u4e00`–`\u9fff`) and trigger translation.

14. **img2img denoise should default to 0.3–0.5**, not 0.75. Denoise controls
    how much the output deviates from the input: 0.2–0.3 = minor style/filter,
    0.4–0.5 = preserve composition but change style, 0.6–0.7 = major changes,
    0.8–0.9 = nearly equivalent to txt2img. Expose as a slider in any UI.

15. **Chinese text on generated images requires post-processing** — SD models
    cannot render CJK characters. Use PIL/Pillow with a CJK font (PingFang on
    macOS, Noto Sans CJK on Linux) to overlay text after generation. See
    aigc-studio `app/postprocess.py` for implementation.

## AnimateDiff Video Workflows

For SD1.5-based AnimateDiff (found under `custom_nodes/ComfyUI-AnimateDiff-Evolved`):

**Motion module:** `mm_sd_v15_v2.ckpt` (1.7GB) — use `.ckpt` format, not `.safetensors`.
Download from `https://huggingface.co/guoyww/animatediff` (VPN required in China).

**Node: AnimateDiffLoaderV1** (V2 does NOT exist):
- Inputs: `model` (MODEL), `latents` (LATENT), `model_name` (string)
- Outputs: `[0]` = MODEL (motion-modified), `[1]` = LATENT (motion-modified)
- KSampler must use: `model: ["aninode", 0]`, `latent_image: ["aninode", 1]`
- Using wrong output indices causes type mismatch errors

**Minimal API-format workflow:**
```json
{"1":{"class_type":"CheckpointLoaderSimple","inputs":{"ckpt_name":"sd_v1-5.safetensors"}},
 "2":{"class_type":"AnimateDiffLoaderV1","inputs":{"model":["1",0],"latents":["4",0],"model_name":"mm_sd_v15_v2.ckpt","unlimited_area_hack":false,"beta_schedule":"sqrt_linear (AnimateDiff)"}},
 "4":{"class_type":"EmptyLatentImage","inputs":{"width":512,"height":512,"batch_size":16}},
 "5":{"class_type":"CLIPTextEncode","inputs":{"text":"PROMPT","clip":["1",1]}},
 "6":{"class_type":"CLIPTextEncode","inputs":{"text":"NEGATIVE","clip":["1",1]}},
 "7":{"class_type":"KSampler","inputs":{"seed":0,"steps":20,"cfg":7.5,"sampler_name":"euler","scheduler":"normal","denoise":1.0,"model":["1",0],"positive":["5",0],"negative":["6",0],"latent_image":["2",1]}},
 "8":{"class_type":"VAEDecode","inputs":{"samples":["7",0],"vae":["1",2]}},
 "9":{"class_type":"VHS_VideoCombine","inputs":{"frame_rate":8,"loop_count":0,"filename_prefix":"anim","format":"image/gif","pingpong":false,"save_output":true,"images":["8",0]}}}
```

**Validation:** submit directly to `POST /prompt` — `node_errors: {}` confirms correct topology.

## HunyuanVideo Workflows

HunyuanVideo 1.5 requires separate model files (NOT a single safetensors checkpoint):
- `models/text_encoders/qwen_2.5_vl_7b_fp8_scaled.safetensors` (~9GB)
- `models/text_encoders/byt5_small_glyphxl_fp16.safetensors` (~0.4GB)
- `models/diffusion_models/hunyuanvideo1.5_720p_t2v_fp16.safetensors` (~16GB)
- `models/vae/hunyuanvideo15_vae_fp16.safetensors` (~2GB)
Total: ~27GB. Source: `https://huggingface.co/Comfy-Org/HunyuanVideo_1.5_repackaged`

**Performance:** M1 Max 32GB: 17 frames × 480p × 10 steps = 5-8 min, 49 frames × 720p = 20+ min.
RTX 4090: 49 frames × 720p = 2-3 min. MPS is 5-10× slower than CUDA for video models.
**Recommendation:** Skip HunyuanVideo on Apple Silicon. Use AnimateDiff SD1.5 for video
or focus on SDXL image generation which runs at acceptable speed (30-60s/image).

**Node: DualCLIPLoader** — type must be `"hunyuan_video_15"` (NOT `"hunyuan_video_1.5"`).
```
Inputs: clip_name1=qwen_2.5_vl_7b_fp8_scaled.safetensors, clip_name2=byt5_small_glyphxl_fp16.safetensors, type="hunyuan_video_15"
```

**Node: VAEDecode (regular, NOT VAEDecodeHunyuan3D)** — VAEDecodeHunyuan3D outputs VOXEL type which is incompatible with image/video output nodes. Use regular VAEDecode with the HunyuanVideo VAE loaded through VAELoader.

**Sampler chain** (NOT a single KSampler):
```
BasicScheduler → RandomNoise → KSamplerSelect → CFGGuider → SamplerCustomAdvanced
```
CFGGuider outputs GUIDER (not MODEL). Connect CFGGuider directly to SamplerCustomAdvanced's `guider` input.

**Output chain:** `VAEDecode → CreateVideo → SaveVideo` (CreateVideo outputs VIDEO type, SaveVideo accepts VIDEO).

**Performance on M1 Max:** 17 frames × 480p × 10 steps takes 5-8 minutes. 49 frames × 720p can exceed 20 minutes.

## Translation Backstop Pattern

When building a wrapper around ComfyUI with Chinese-language UI:

**Anti-pattern:** translation ONLY in the API endpoint layer. If the endpoint fails silently, Chinese text reaches SDXL directly (= random noise).

**Correct pattern:** translation as the FIRST step INSIDE the engine function:
```python
async def generate_video(prompt, ...):
    # ⚠️ Must be inside engine, not just endpoint
    if any('\u4e00' <= c <= '\u9fff' for c in prompt):
        translated = await llm_translate(prompt)
        if translated and no_chinese(translated):
            prompt = translated
    # ... submit to ComfyUI
```

## Verification Checklist

Use `python3 scripts/health_check.py` to run the whole list at once. Manual:

- [ ] `hardware_check.py` verdict is `ok` OR the user explicitly chose Comfy Cloud
- [ ] `comfy --version` works (or `uvx --from comfy-cli comfy --help`)
- [ ] `curl http://HOST:PORT/system_stats` returns JSON
- [ ] `comfy model list` shows at least one checkpoint (local) OR
      `/api/experiment/models/checkpoints` returns models (cloud)
- [ ] Workflow JSON is in API format
- [ ] `check_deps.py` reports `is_ready: true` (or only `node_check_skipped`
      on cloud free tier)
- [ ] If using SDXL: workflow uses `CLIPTextEncodeSDXL` (not `CLIPTextEncode`)
- [ ] If user inputs Chinese: prompt is auto-translated to English before submission
