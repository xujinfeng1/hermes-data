# AIGC Studio Reference Architecture

Built during 2026-05-24 session. A web frontend + FastAPI backend that wraps
ComfyUI for text-to-image / text-to-video generation with LLM prompt optimization.

## Key Components

```
aigc-studio/
├── app/
│   ├── main.py         # FastAPI (port 8900) — /generate, /generate-video, /optimize
│   ├── comfyui.py      # ComfyUI REST client — submit_workflow, wait_for_result, download_outputs
│   ├── workflows.py    # Workflow templates + param injection
│   ├── prompt_engine.py # DeepSeek prompt optimization (CN→EN)
│   ├── postprocess.py  # PIL text overlay for Chinese text on images
│   └── video_gen.py    # AnimateDiff video generation
├── index.html          # Web UI (industry-standard layout: controls left, output right)
├── workflows/          # ComfyUI workflow JSON files
├── outputs/            # Generated images/videos
└── requirements.txt
```

## ComfyUI Workflow Integration

- Submit: `POST /api/prompt` with API-format workflow JSON
- Poll: `GET /api/history/{prompt_id}` until status is "success"
- Download: `GET /api/view?filename=X&type=output`
- Upload img2img: `POST /api/upload/image` (multipart)

## SDXL Workflow Template (API format)

```python
TXT2IMG_MINIMAL = {
    "3": {"class_type": "KSampler", "inputs": {"seed": 0, "steps": 30, "cfg": 8.0,
        "sampler_name": "dpmpp_2m", "scheduler": "karras", ...}},
    "4": {"class_type": "CheckpointLoaderSimple", "inputs": {"ckpt_name": "sd_xl_base_1.0.safetensors"}},
    "6": {"class_type": "CLIPTextEncodeSDXL", "inputs": {"text_g": "PROMPT_PLACEHOLDER",
        "text_l": "PROMPT_PLACEHOLDER", ...}},  # CRITICAL: must use SDXL variant
    "9": {"class_type": "SaveImage", "inputs": {"filename_prefix": "aigc", ...}},
}
```

## Hardware Reference

- M1 Max 32GB: SDXL ~48s/image at 1024x1024
- SD1.5: ~15s/image at 512x512
- AnimateDiff SD1.5: ~3min/16-frame video at 512x512

## LLM Prompt Optimization

- Input: Chinese description → DeepSeek V4 Pro → English SD prompt
- Response format: `{"prompt": "...", "negative_prompt": "..."}`
- Auto-detection: `any('\u4e00' <= c <= '\u9fff' for c in prompt)` triggers translation
