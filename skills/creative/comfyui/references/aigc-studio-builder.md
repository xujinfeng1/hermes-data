# AIGC Studio Builder Patterns

## Critical: Chinese → English Translation

**Rule: translate at the INNERMOST data entry point, never only at the outer endpoint.**

Wrong pattern (fixed 4 times, failed each time):
```python
# main.py endpoint
@app.post("/generate-video")
async def generate_video(req):
    prompt = translate(req.prompt)  # ❌ endpoint translation
    asyncio.create_task(_generate_video_async(prompt=prompt))
    # Problem: if _generate_video_async calls generate_animated_video directly,
    # and someone adds a new call site, Chinese leaks through.

async def _generate_video_async(job_id, prompt, ...):
    result = await generate_animated_video(prompt=prompt)
    # If prompt wasn't translated upstream, Chinese hits SDXL = garbage
```

Correct pattern:
```python
# video_gen.py engine function
async def generate_animated_video(prompt: str, ...):
    # ✅ translation at the innermost data entry point
    if any('\u4e00' <= c <= '\u9fff' for c in prompt):
        result = await optimize_prompt(prompt)
        if result.get("prompt") and not any('\u4e00' <= c <= '\u9fff' for c in result["prompt"]):
            prompt = result["prompt"]
    # Now safe — no Chinese can reach ComfyUI from here
    workflow = apply_params(workflow, {"prompt": prompt})
```

Also keep the endpoint translation as convenience, but never rely on it as the only defense.

## ComfyUI Node Selection

### AnimateDiff
- Current installation: `AnimateDiffLoaderV1` (evolved), NOT `AnimateDiffLoaderV2` or `ADE_*`
- Inputs: model (MODEL), latents (LATENT), model_name (string)
- Output [0] = MODEL (motion-modified), output [1] = LATENT
- KSampler wiring: model=[animdiff_node, 0], latent_image=[animdiff_node, 1]

### HunyuanVideo 1.5
- Model: HunyuanVideo 1.5 repackaged (27GB, 4 files)
- DualCLIPLoader: type="hunyuan_video_15"
- **Use regular `VAEDecode`**, NOT `VAEDecodeHunyuan3D` (that outputs VOXEL)
- Sampler chain (not simple KSampler): BasicScheduler → RandomNoise → KSamplerSelect → CFGGuider → SamplerCustomAdvanced
- Optimize: 17 frames @ 480p @ 10 steps for testing (~5min), 49 frames @ 720p for production (~15min)
- Save chain: VAEDecode → CreateVideo → SaveVideo

### SDXL
- Must use `CLIPTextEncodeSDXL` (dual CLIP), not `CLIPTextEncode`
- CFG ≥ 8 for prompt adherence

## Mac Apple Silicon Specifics

- ComfyUI startup: use direct python, not comfy-cli (`comfy launch` may corrupt venv)
  ```bash
  cd ~/Documents/comfy/ComfyUI && .venv/bin/python main.py --enable-manager
  ```
- MPS memory fragmentation after ~10-20 generations → ComfyUI stops responding
  - Fix: `pkill -9 -f "ComfyUI/main"` then restart
  - Schedule periodic restart in production
- Model warmup: first SDXL run takes ~60s, subsequent ~30s

## Network (China)

- Terminal `curl`/`python` DO NOT route through system VPN on macOS
- Model downloads: use browser + VPN, place files manually
- Download priority: modelscope.cn > hf-mirror.com > ghproxy.net > direct HF (VPN)
- Motion modules (AnimateDiff) NOT available on any Chinese mirror