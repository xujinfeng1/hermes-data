# SDXL on Mac (Apple Silicon) — Workflow Pitfalls

## Critical: SDXL Requires CLIPTextEncodeSDXL

SDXL has dual CLIP text encoders (CLIP-L + CLIP-G). Using plain `CLIPTextEncode` 
will silently produce outputs entirely unrelated to the prompt.

**Correct:** `CLIPTextEncodeSDXL` node with both `text_g` and `text_l` filled.
**Wrong:** `CLIPTextEncode` (single encoder — image looks like random noise vs prompt).

## SDXL Prompt Language

SDXL only understands English. Chinese prompts produce garbage.

**Fix:** Auto-translate via LLM before injecting into workflow:
```python
if any('\u4e00' <= c <= '\u9fff' for c in prompt):
    result = await optimize_prompt(prompt)  # DeepSeek LLM
    prompt = result["prompt"]  # English
```

## Recommended SDXL Settings for M1 Max

```
Sampler: dpmpp_2m + karras
Steps: 30 (quality) or 20 (speed)
CFG: 8.0 (lower = less prompt-following)
Resolution: 1024x1024 native, up to 1344x768 safe
```

## MPS Memory Issues

ComfyUI on Mac MPS can hang after several hours of generation. 
Symptoms: HTTP requests to :8188 timeout, process still running, GPU memory fragmentation.
Fix: kill process, `comfy launch --background` restart.

## Dependency: sqlalchemy

Recent ComfyUI versions require sqlalchemy. If missing:
```
/Users/xu/Documents/comfy/ComfyUI/.venv/bin/pip install sqlalchemy
```

## Video: AnimateDiff Required

Generating frames with different seeds and stitching = completely incoherent video.
For real video: install `comfyui-animatediff-evolved` and `comfyui-videohelpersuite`,
then download a motion module model.

## Workflow JSON — API Format

Workflows submitted to /api/prompt must be API format (each node has `class_type`).
Editor format (top-level `nodes` and `links` arrays) is NOT executable.
Export via ComfyUI UI → Workflow → Export (API).
