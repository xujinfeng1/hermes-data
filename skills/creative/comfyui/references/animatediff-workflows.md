# AnimateDiff Video Generation — Workflow Patterns

## Node Names (AnimateDiff Evolved)

The `comfyui-animatediff-evolved` package provides ADE-prefixed nodes.
Common mistake: using `AnimateDiffLoaderV2` which doesn't exist.

### Working SD1.5 Video Workflow

```json
{
    "1": {"class_type": "CheckpointLoaderSimple", "inputs": {"ckpt_name": "sd_v1-5.safetensors"}},
    "2": {"class_type": "ADE_LoadAnimateDiffModel", "inputs": {"model_name": "mm_sd_v15_v2.ckpt"}},
    "3": {"class_type": "ADE_ApplyAnimateDiffModelSimple", "inputs": {"motion_model": ["2", 0], "model": ["1", 0]}},
    "4": {"class_type": "EmptyLatentImage", "inputs": {"width": 512, "height": 512, "batch_size": 16}},
    "5": {"class_type": "CLIPTextEncode", "inputs": {"text": "PROMPT", "clip": ["1", 1]}},
    "6": {"class_type": "CLIPTextEncode", "inputs": {"text": "NEGATIVE", "clip": ["1", 1]}},
    "7": {"class_type": "KSampler", "inputs": {"seed": 0, "steps": 20, "cfg": 7.5, "sampler_name": "euler", "scheduler": "normal", "denoise": 1.0, "model": ["3", 0], "positive": ["5", 0], "negative": ["6", 0], "latent_image": ["4", 0]}},
    "8": {"class_type": "VAEDecode", "inputs": {"samples": ["7", 0], "vae": ["1", 2]}},
    "9": {"class_type": "VHS_VideoCombine", "inputs": {"frame_rate": 8, "loop_count": 0, "filename_prefix": "aigc_anim", "format": "image/gif", "pingpong": false, "save_output": true, "images": ["8", 0]}}
}
```

Topology: Checkpoint → ADE_LoadModel → ADE_ApplyModel → KSampler → VAEDecode → VHS_VideoCombine

### Key differences from image generation:
- Motion model goes into `ADE_LoadAnimateDiffModel` (not the checkpoint)
- `ADE_ApplyAnimateDiffModelSimple` wraps the checkpoint model with motion
- KSampler takes the APPLIED model, not the raw checkpoint
- `EmptyLatentImage.batch_size` = number of frames (not 1)
- `VHS_VideoCombine` is the output node (not `SaveImage`)
- On M1 Max 32GB: 16 frames × SD1.5 ≈ 5-8 minutes

## Motion Module Files

- `.ckpt` files work (original PyTorch format)
- `.safetensors` also supported but may need naming
- File goes in: `ComfyUI/models/animatediff_models/`
- SD1.5: `mm_sd_v15_v2.ckpt` (1.7GB)
- SDXL: `mm_sdxl_v10.safetensors` (708MB) — less stable

## Download Challenges (China)

All Chinese mirrors fail for motion modules. Only options:
1. VPN + browser download from huggingface.co
2. Direct download file to `ComfyUI/models/animatediff_models/`
3. Then restart ComfyUI to pick up the file

## ComfyUI Stability on Apple Silicon

- After hours of runtime, ComfyUI may hang (MPS memory fragmentation)
- Symptoms: process alive, using RAM, but HTTP times out
- Fix: kill process, ensure sqlalchemy installed in venv, relaunch
- Use direct `python main.py --enable-manager` instead of `comfy launch --background` if launch fails
