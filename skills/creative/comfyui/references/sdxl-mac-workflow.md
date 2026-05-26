# SDXL on Apple Silicon — Workflow Gotchas

## Critical: CLIPTextEncodeSDXL (NOT CLIPTextEncode)

SDXL uses dual CLIP encoders (CLIP-L + CLIP-G). Using plain `CLIPTextEncode`
produces images completely unrelated to the prompt.

```json
{
  "class_type": "CLIPTextEncodeSDXL",
  "inputs": {
    "width": 1024, "height": 1024,
    "crop_w": 0, "crop_h": 0,
    "target_width": 1024, "target_height": 1024,
    "text_g": "PROMPT_PLACEHOLDER",
    "text_l": "PROMPT_PLACEHOLDER",
    "clip": ["4", 1]
  }
}
```

Both `text_g` and `text_l` MUST be set to the same prompt. Use
`CheckpointLoaderSimple` which auto-provides dual CLIP for SDXL checkpoints.

## M1 Max Optimized Parameters

| Param | Value | Why |
|-------|-------|-----|
| sampler | dpmpp_2m | Best quality/speed on MPS |
| scheduler | karras | Better contrast |
| steps | 30 | SDXL needs more than SD1.5 |
| CFG | 8.0 | Stronger prompt adherence |
| resolution | 1024x1024 | SDXL native |

## Chinese Prompts

SDXL training data is English-only. Chinese prompts produce garbage.
Auto-translate with LLM before sending to ComfyUI:

```python
if any('\u4e00' <= c <= '\u9fff' for c in prompt):
    prompt = await llm.translate_to_english(prompt)
```

## Model Downloads in China

HuggingFace is blocked. Use hf-mirror.com:
```bash
curl -L -o model.safetensors \
  "https://hf-mirror.com/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors"
```

## Stability on Mac

ComfyUI can hang after hours of use due to MPS memory fragmentation.
Restart periodically: `kill <pid> && comfy launch --background`

## Img2Img Denoise

Default 0.5 preserves original image. Range: 0.2 (barely changes) to 0.9 (near txt2img).

## Video Generation

Generate frames sequentially with different seeds, stitch to GIF with PIL.
Submit async (return job_id immediately), poll for completion.
Never block HTTP response — 8 frames × 30s = 4+ minutes.
