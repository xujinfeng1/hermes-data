# AIGC Studio: FastAPI Web Wrapper for ComfyUI

## Architecture
```
Browser → FastAPI (8900) → ComfyUI REST API (8188) → SDXL/SD1.5
                ↓
          DeepSeek LLM (prompt optimization + Chinese→English)
```

## Key Components

### ComfyUI Client (`app/comfyui.py`)
- `ComfyClient.submit_workflow(workflow, client_id)` → `{prompt_id, number}`
- `ComfyClient.wait_for_result(prompt_id, timeout)` — polling loop
- `ComfyClient.download_outputs(history)` — handles both `{images:[{filename}]}` and `{gifs:[{filename}]}` formats
- `ComfyClient.upload_image(file_path)` — for img2img

### Workflow Templates (`app/workflows.py`)
- `apply_params(workflow, params)` injects prompt/seed/steps/size into workflow JSON
- Supports both `CLIPTextEncode` (SD1.5) and `CLIPTextEncodeSDXL` (SDXL)
- Template types: txt2img, img2img, txt2video (AnimateDiff)

### LLM Prompt Engine (`app/prompt_engine.py`)
- `optimize_prompt(chinese_text)` → DeepSeek translates to English SD prompt
- `generate_inspiration(topic, count)` → creative prompt ideas
- Auto-detection: `any('\u4e00' <= c <= '\u9fff' for c in text)`

### Async Job System
- `/generate` returns `{job_id, status: "running"}` immediately
- Background `asyncio.create_task` handles the slow ComfyUI work
- `/jobs/{id}` for polling status — frontend polls every 2-3 seconds
- Jobs stored in `jobs: dict[str, dict]` (in-memory)

### Chinese Prompt Handling
```
User types Chinese → `/generate` detects CJK chars → `optimize_prompt()` → 
DeepSeek translates → English prompt sent to ComfyUI
```
This MUST happen on the backend. SDXL/SD1.5 cannot process Chinese.

### Post-Processing (`app/postprocess.py`)
- `overlay_text(image_path, text)` — PIL-based Chinese text overlay
- Uses PingFang/STHeiti fonts on macOS

## Web UI Patterns
- `onclick="fn(event)"` — MUST pass `event` explicitly to handlers that need it
- `<input type="range">` for denoise/frames sliders
- Canvas-style aspect ratio buttons (`1:1 3:2 16:9`) instead of number inputs
- Collapsible advanced settings panel (`⚙️ 高级设置 ▾`)
- Tab separation: Image / Video as separate modes (industry standard: Leonardo/Runway)

## Common Issues
- **"生成内容完全不相关"**: Chinese prompt not translated, or CLIPTextEncode used instead of CLIPTextEncodeSDXL
- **图生图卡住**: Image file not uploaded to ComfyUI input directory first
- **生图能行生视频不行**: AnimateDiff nodes/model not installed
- **ComfyUI health check returns offline**: Process stuck, need restart
