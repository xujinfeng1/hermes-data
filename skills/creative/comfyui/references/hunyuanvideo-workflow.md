# HunyuanVideo 1.5 Workflow Construction

## Model Requirements (~27GB total)

| Model | Size | Location |
|-------|------|----------|
| qwen_2.5_vl_7b_fp8_scaled.safetensors | 8.7GB | models/text_encoders/ |
| byt5_small_glyphxl_fp16.safetensors | 418MB | models/text_encoders/ |
| hunyuanvideo1.5_720p_t2v_fp16.safetensors | 16GB | models/diffusion_models/ |
| hunyuanvideo15_vae_fp16.safetensors | 2.3GB | models/vae/ |

Source: `https://huggingface.co/Comfy-Org/HunyuanVideo_1.5_repackaged/tree/main/split_files`

## Sampler Chain (DIFFERENT from standard KSampler)

HunyuanVideo uses a complex sampler chain instead of simple KSampler:

```
DualCLIPLoader → CLIPTextEncode (positive & negative)
UNETLoader → BasicScheduler → CFGGuider → SamplerCustomAdvanced
VAELoader → VAEDecode → CreateVideo → SaveVideo
RandomNoise + KSamplerSelect → SamplerCustomAdvanced
EmptyHunyuanVideo15Latent → SamplerCustomAdvanced
```

Key node types and connections:
- `DualCLIPLoader` type MUST be `"hunyuan_video_15"` (not `"hunyuan_video_1.5"`)
- `DualCLIPLoader` outputs 2 clips; use `["1", 0]` for CLIPTextEncode
- `SamplerCustomAdvanced` needs: noise, guider, sampler, sigmas, latent_image
- `CFGGuider` outputs GUIDER, feeds into SamplerCustomAdvanced guider input
- `CreateVideo` + `SaveVideo` for output (NOT VHS_VideoCombine)
- Frame count controlled by `EmptyHunyuanVideo15Latent.length` (17=fast, 49=default, 121=long)

## Minimal Working Workflow (API format)

```json
{
  "1": {"class_type": "DualCLIPLoader", "inputs": {
    "clip_name1": "qwen_2.5_vl_7b_fp8_scaled.safetensors",
    "clip_name2": "byt5_small_glyphxl_fp16.safetensors",
    "type": "hunyuan_video_15"}},
  "2": {"class_type": "UNETLoader", "inputs": {
    "unet_name": "hunyuanvideo1.5_720p_t2v_fp16.safetensors",
    "weight_dtype": "default"}},
  "3": {"class_type": "VAELoader", "inputs": {
    "vae_name": "hunyuanvideo15_vae_fp16.safetensors"}},
  "4": {"class_type": "CLIPTextEncode", "inputs": {
    "text": "PROMPT", "clip": ["1", 0]}},
  "5": {"class_type": "CLIPTextEncode", "inputs": {
    "text": "NEGATIVE", "clip": ["1", 0]}},
  "6": {"class_type": "EmptyHunyuanVideo15Latent", "inputs": {
    "width": 480, "height": 320, "length": 17, "batch_size": 1}},
  "7": {"class_type": "BasicScheduler", "inputs": {
    "model": ["2", 0], "scheduler": "simple", "steps": 10, "denoise": 1.0}},
  "8": {"class_type": "RandomNoise", "inputs": {"noise_seed": 42}},
  "9": {"class_type": "KSamplerSelect", "inputs": {"sampler_name": "euler"}},
  "10": {"class_type": "CFGGuider", "inputs": {
    "model": ["2", 0], "positive": ["4", 0], "negative": ["5", 0], "cfg": 6.0}},
  "11": {"class_type": "SamplerCustomAdvanced", "inputs": {
    "noise": ["8", 0], "guider": ["10", 0], "sampler": ["9", 0],
    "sigmas": ["7", 0], "latent_image": ["6", 0]}},
  "12": {"class_type": "VAEDecode", "inputs": {
    "samples": ["11", 0], "vae": ["3", 0]}},
  "13": {"class_type": "CreateVideo", "inputs": {
    "images": ["12", 0], "fps": 12}},
  "14": {"class_type": "SaveVideo", "inputs": {
    "video": ["13", 0], "filename_prefix": "hunyuan",
    "format": "auto", "codec": "h264"}}
}
```

## Performance on M1 Max 32GB

- 49 frames × 720p × 15 steps: ~15-20 minutes
- 17 frames × 480p × 10 steps: ~5-8 minutes (lightweight test)
- Memory: requires ~20GB unified memory at peak

## Pitfalls

1. **DualCLIPLoader type** — must be `"hunyuan_video_15"`, NOT `"hunyuan_video_1.5"` (causes `value_not_in_list` error)
2. **VAEDecodeHunyuan3D outputs VOXEL** — don't use it; use regular `VAEDecode` with the Hunyuan VAE loaded via `VAELoader`
3. **CLIPTextEncodeHunyuanDiT** — requires `bert` and `mt5xl` inputs that are tricky to wire; use regular `CLIPTextEncode` with `clip: ["1", 0]` instead
4. **Output chain** — must be `CreateVideo → SaveVideo`, not `VHS_VideoCombine` (type mismatch)
5. **No negative prompt node** — HunyuanVideo doesn't support negative prompt well; use same clip for both or omit
6. **Apple Silicon** — first run loads all 27GB into unified memory; subsequent runs faster due to caching
