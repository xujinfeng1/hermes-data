# SDXL Workflow Configuration

## Critical: Use CLIPTextEncodeSDXL

SDXL has **dual text encoders**: CLIP-L (ViT-L) and CLIP-G (ViT-bigG).
Using the single `CLIPTextEncode` node causes completely unrelated images
because only CLIP-L is used, losing ~50% of semantic understanding.

### Correct node configuration:

```json
{
  "class_type": "CLIPTextEncodeSDXL",
  "inputs": {
    "width": 1024, "height": 1024,
    "crop_w": 0, "crop_h": 0,
    "target_width": 1024, "target_height": 1024,
    "text_g": "YOUR POSITIVE PROMPT",
    "text_l": "YOUR POSITIVE PROMPT",
    "clip": ["4", 1]
  }
}
```

Both `text_g` and `text_l` should contain the same prompt for basic usage.

## Recommended Settings for M1 Max (MPS)

| Parameter | Value |
|-----------|-------|
| Steps | 20-30 |
| CFG Scale | 8.0 |
| Sampler | dpmpp_2m |
| Scheduler | karras |
| Resolution | 1024x1024 (native) |
| Batch size | 1 |

Generation time: ~50s per 1024x1024 image on M1 Max 32GB.
SDXL trained at 1024x1024. Other common resolutions: 896x1152, 1152x896, 768x1344.
