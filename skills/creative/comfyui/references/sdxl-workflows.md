# SDXL Workflow Templates (API Format)

## Minimal txt2img (SDXL)

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

## Key SDXL Parameters

| Param | Default | Range | Notes |
|-------|---------|-------|-------|
| steps | 30 | 20-50 | Higher = more detail, slower |
| cfg | 8.0 | 7-10 | Higher = stronger prompt following |
| sampler | dpmpp_2m | - | karras scheduler recommended |
| width/height | 1024 | 768-1536 | SDXL native is 1024^2 |

## Img2img Workflow

Same as above but add LoadImage → VAEEncode between latent and sampler:
- Node 10: VAEEncode, inputs: {"pixels": ["11", 0], "vae": ["4", 2]}
- Node 11: LoadImage, inputs: {"image": "UPLOADED_FILENAME"}
- KSampler denoise: 0.4-0.5 for style transfer, 0.2-0.3 for filter only

Upload flow:
1. POST file to /upload/image (multipart form)
2. Get {"name": "file.png"} back
3. Inject filename into LoadImage node

## Video via Multi-Frame

Generate N frames (4-16) with different random seeds, same prompt.
Download each frame, stitch with PIL:
```python
frames[0].save('output.gif', save_all=True, append_images=frames[1:], duration=250, loop=0)
```
No custom ComfyUI nodes needed.
