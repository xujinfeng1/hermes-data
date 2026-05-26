---
name: aigc-studio
description: "Build a FastAPI web wrapper around ComfyUI for text-to-image with auto prompt translation, Chinese text overlay, and web UI."
version: 1.2.0
author: hermes
platforms: [macos, linux]
compatibility: "Requires ComfyUI (local) on :8188, DeepSeek API key, PIL/Pillow."
metadata:
  hermes:
    tags: [aigc, comfyui, fastapi, text-to-image, web-ui]
    related_skills: [comfyui]
    category: creative
---

# AIGC Studio

FastAPI wrapper around ComfyUI with prompt translation, text overlay, and web UI.

## Project Structure

```
aigc-studio/
├── app/
│   ├── main.py            # FastAPI (port 8900)
│   ├── comfyui.py         # ComfyUI REST client
│   ├── workflows.py       # Templates + param injection
│   ├── prompt_engine.py   # DeepSeek CN→EN translation
│   ├── postprocess.py     # PIL Chinese text overlay
│   └── video_gen.py       # Video generation (AnimateDiff)
├── index.html             # Web UI
├── workflows/             # ComfyUI workflow JSON
└── outputs/               # Generated images
```

## Industry Standard UX

Based on Leonardo.ai, RunwayML, Kling:

- Separate tabs for image vs video (don't mix params)
- Visual aspect ratio selector (1:1/3:2/16:9 buttons) — not number inputs
- Negative prompt collapsed behind toggle (+)
- Advanced settings collapsed (gear icon: steps, seed, CFG)
- Left panel = controls, Right panel = output
- Gallery grid + lightbox for browsing
- Prompt optimization button next to input

## Key Decisions

### Auto-translate Chinese to English

SDXL cannot understand Chinese. Detect and auto-translate before submitting:

```python
if any('\u4e00' <= c <= '\u9fff' for c in prompt):
    result = await optimize_prompt(prompt)
    if not any('\u4e00' <= c <= '\u9fff' for c in result["prompt"]):
        prompt = result["prompt"]
```

### CLIPTextEncodeSDXL, NOT CLIPTextEncode

SDXL uses dual text encoders (CLIP-L + CLIP-G). Plain CLIPTextEncode causes
completely unrelated output. Always use CLIPTextEncodeSDXL with text_g + text_l.

### Img2img upload flow

Browser reads file as base64 → POST /generate → backend decodes → uploads
to ComfyUI /upload/image → injects filename into workflow.

### Img2img denoise default: 0.5

Not 0.75. Lower = preserve original. Expose slider with label "越低越保留原图".

### Post-gen Chinese text overlay

SD models can't render CJK. Use PIL + PingFang font via POST /text-overlay.

## Video Generation

### AnimateDiff (coherent video — preferred)

Requires: comfyui-animatediff-evolved, comfyui-videohelpersuite, motion module (~708MB).

In China, HuggingFace is blocked. hf-mirror has checkpoints but NOT motion modules.
Workaround: download SD1.5 model + its motion module from hf-mirror (~6GB), or use VPN.

### Frame-by-frame (DO NOT use)

Generating frames with different seeds = unrelated images. Each frame has no
concept of the previous one. Always use AnimateDiff for temporal coherence.

### Async job pattern

Video takes 3-5 minutes. Return job_id immediately, poll /jobs/{id} every 3s.

## ComfyUI Reliability

### Process hangs on Apple Silicon (MPS)

After hours, ComfyUI freezes (process alive, port open, HTTP times out).
Cause: GPU memory fragmentation. Fix: kill process and restart.

### Restart may expose missing deps

Killing a long-running process → restart fails with ModuleNotFoundError
(e.g. sqlalchemy). Old process had them in memory. Reinstall in the
ComfyUI venv: `pip install <package>`.

### Always check ComfyUI health FIRST

When generation fails — `curl http://127.0.0.1:8188/system_stats` before
debugging app code. ComfyUI being down is the #1 cause of all failures.

## Pitfalls

See `references/sdxl-troubleshooting.md` for diagnosis flow.

1. SDXL needs CLIPTextEncodeSDXL — plain CLIPTextEncode = unrelated images
2. Chinese prompts → always auto-translate INSIDE engine function (not just endpoint)
3. img2img must upload to ComfyUI first — missing upload = hung forever
4. ComfyUI can freeze on MPS after hours — restart needed
5. M1 Max 32GB: SDXL 1024×1024 @ 30 steps ~45s; video impractical
6. History outputs are {"<node_id>": {"images": [...]}} — unwrap before iterating
7. HF downloads in China: modelscope.cn > hf-mirror.com > VPN browser
8. Denoise default 0.5 (preserves original), not 0.75
9. Always diagnose ComfyUI before debugging app code
10. JS syntax: verify with `node -e "new Function(js)"` after EVERY HTML edit
11. HTML patching: grep for duplicate functions and orphaned keywords after edits
12. `event` in onclick handlers: pass as parameter `onclick="fn(event)"`, use `e?.target`