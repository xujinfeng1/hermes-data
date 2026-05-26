# AIGC Studio: Web Wrapper + API Gateway Patterns

## Architecture

```
Browser → AIGC Studio (FastAPI :8900) → ComfyUI (:8188)
                │
                ├── Prompt Engine (DeepSeek LLM)
                │     └── 中文→英文 automatic translation
                ├── Workflow Templates (txt2img/img2img/video)
                └── Job Queue (async polling)
```

## Key Patterns

### 1. Chinese Prompt Auto-Translation
SD/SDXL models do NOT understand Chinese. Every prompt must be English.
- **Detection**: `any('\u4e00' <= c <= '\u9fff' for c in prompt)`
- **Translation**: DeepSeek API → professional English SD prompt + negative
- **Verification**: Check output has no Chinese chars before sending to ComfyUI
- **Where**: Translate INSIDE the generation function, not just at the endpoint.
  Translation at API boundary can fail silently. Translate as close to model as possible.

### 2. Async Job Polling
Video generation takes 3-10 minutes. Never block HTTP response.
- Return `job_id` immediately, create background task
- Frontend polls `/jobs/{id}` every 2-3s
- Show progress bar, then result when completed

### 3. Workflow Parameter Injection
Use placeholder strings in workflow JSON, replace at submission time:
- `PROMPT_PLACEHOLDER` → actual prompt
- `NEGATIVE_PLACEHOLDER` → negative prompt
- Handle both `CLIPTextEncode` (text field) and `CLIPTextEncodeSDXL` (text_g + text_l)

### 4. Image Upload for img2img
- Frontend reads file as base64 via FileReader
- Backend decodes base64, writes temp file, uploads to ComfyUI `/upload/image`
- Use returned filename in workflow's LoadImage node

## Pitfalls

### ComfyUI Dependency Issues
- `sqlalchemy` may be missing after fresh install → install in venv
- `comfy launch --background` may fail while direct `python main.py` works
- Always verify with `curl localhost:8188/system_stats`

### Model Downloads in China
- huggingface.co: BLOCKED, needs VPN
- hf-mirror.com: unreliable, may timeout
- modelscope.cn: works for SD1.5, SDXL. Does NOT host motion modules
- Browser VPN: most reliable for HF-only files
- Terminal curl: does NOT route through system VPN on macOS

### AnimateDiff Workflow Debugging
- `AnimateDiffLoaderV2` may not exist → use `AnimateDiffLoaderV1`
- Motion module: `.ckpt` and `.safetensors` are both supported
- KSampler `latent_image`: must connect to loader output [1] (LATENT), not [0] (MODEL)
- VHS_VideoCombine: requires `pingpong` parameter
- ADE nodes (AnimateDiff Evolved) use different topology than V1 loader

### HunyuanVideo Complexity
- Requires 4 model files (~27GB): VAE, diffusion, 2 text encoders
- Workflow uses `SamplerCustomAdvanced` + `CFGGuider`, NOT `KSampler`
- DualCLIPLoader type is "hunyuan_video_15" (underscore, not dot)
- CLIPTextEncodeHunyuanDiT needs `bert` and `mt5xl` inputs from DualCLIPLoader
- Output: VAEDecode → CreateVideo → SaveVideo chain
- SaveVideo alone doesn't register as output node → "Prompt has no outputs" error
  Must include CreateVideo as intermediate node

## Text Overlay
- PIL + PingFang font for Chinese text on generated images
- Post-processing step, separate from generation
- Font paths differ by OS (PingFang.ttc on macOS)
