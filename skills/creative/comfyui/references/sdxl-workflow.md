# SDXL txt2img Workflow (API Format)

The minimal, correct workflow for SDXL text-to-image generation.

## Critical: CLIPTextEncodeSDXL

SDXL uses **dual text encoders** (CLIP-L + CLIP-G). Using the single `CLIPTextEncode` node produces COMPLETELY UNRELATED images because only CLIP-L is utilized. Always use `CLIPTextEncodeSDXL` with both `text_g` and `text_l` inputs.

## Optimal Settings

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sampler | `dpmpp_2m` | Best quality for SDXL |
| Scheduler | `karras` | Pairs with dpmpp_2m |
| CFG | 8.0 | Higher than SD1.5; controls prompt adherence |
| Steps | 25-30 | SDXL converges slower than SD1.5 |
| Resolution | 1024×1024 | SDXL native resolution |

## Full Workflow JSON

```json
{
  "3": {"class_type": "KSampler", "inputs": {"seed": 0, "steps": 30, "cfg": 8.0, "sampler_name": "dpmpp_2m", "scheduler": "karras", "denoise": 1.0, "model": ["4", 0], "positive": ["6", 0], "negative": ["7", 0], "latent_image": ["5", 0]}},
  "4": {"class_type": "CheckpointLoaderSimple", "inputs": {"ckpt_name": "sd_xl_base_1.0.safetensors"}},
  "5": {"class_type": "EmptyLatentImage", "inputs": {"width": 1024, "height": 1024, "batch_size": 1}},
  "6": {"class_type": "CLIPTextEncodeSDXL", "inputs": {"width": 1024, "height": 1024, "crop_w": 0, "crop_h": 0, "target_width": 1024, "target_height": 1024, "text_g": "PROMPT_PLACEHOLDER", "text_l": "PROMPT_PLACEHOLDER", "clip": ["4", 1]}},
  "7": {"class_type": "CLIPTextEncodeSDXL", "inputs": {"width": 1024, "height": 1024, "crop_w": 0, "crop_h": 0, "target_width": 1024, "target_height": 1024, "text_g": "NEGATIVE_PLACEHOLDER", "text_l": "NEGATIVE_PLACEHOLDER", "clip": ["4", 1]}},
  "8": {"class_type": "VAEDecode", "inputs": {"samples": ["3", 0], "vae": ["4", 2]}},
  "9": {"class_type": "SaveImage", "inputs": {"filename_prefix": "aigc", "images": ["8", 0]}}
}
```

## Param Injection

When injecting parameters programmatically, replace:
- `PROMPT_PLACEHOLDER` with the actual prompt (English only!)
- `NEGATIVE_PLACEHOLDER` with negative prompt
- KSampler seed/steps/cfg values
- EmptyLatentImage width/height

Both `text_g` and `text_l` should receive the same prompt text for standard SDXL.

## Common Pitfall: Chinese Prompts

SDXL was trained on English captions. Chinese text produces random/unrelated images. Always translate to English before submitting. Auto-detect Chinese characters with:
```python
any('\u4e00' <= c <= '\u9fff' for c in prompt)
```
