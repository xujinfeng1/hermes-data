# SDXL Workflow Troubleshooting

## Diagnosis Order (ALWAYS follow this)

When AIGC generation fails, check in this order BEFORE modifying code:

1. `curl http://127.0.0.1:8188/system_stats` — is ComfyUI alive?
2. `curl http://127.0.0.1:8188/queue` — is there a stuck job?
3. `ps aux | grep comfy` — is process hung? (MPS fragmentation)
4. Check history for actual prompt sent: `curl http://127.0.0.1:8188/history`

## Common Issues

### Image unrelated to prompt
- Cause 1: Using CLIPTextEncode instead of CLIPTextEncodeSDXL
- Cause 2: Chinese text sent to SDXL (must be English)
- Fix: CLIPTextEncodeSDXL with text_g + text_l, auto-translate CJK

### Generation stuck
- Cause: ComfyUI frozen from long uptime (MPS memory fragmentation)
- Fix: Kill and restart ComfyUI process
- Follow-up: Restart may fail with missing deps (e.g. sqlalchemy)

### ComfyUI restart fails
- Symptom: ModuleNotFoundError after killing old process
- Cause: Deps were in memory of old process, not persisted/pinned
- Fix: Install in ComfyUI venv: `.venv/bin/pip install <package>`

### img2img hangs
- Cause: Image not uploaded to ComfyUI before workflow submission
- Fix: POST /upload/image first, inject returned filename into LoadImage node

### Video frames unrelated
- Cause: Frame-by-frame generation with different seeds
- Fix: Must use AnimateDiff with motion module for temporal coherence
- Motion modules NOT available on hf-mirror.com in China

## Working SDXL Settings

```
Checkpoint: sd_xl_base_1.0.safetensors
Encoder: CLIPTextEncodeSDXL (NOT CLIPTextEncode)
Sampler: dpmpp_2m + karras
Steps: 30
CFG: 8.0
Resolution: 1024×1024 native
```
