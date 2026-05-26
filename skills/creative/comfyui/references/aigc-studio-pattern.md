# AIGC Studio — ComfyUI Web Wrapper Patterns

## Architecture

```
Browser → AIGC Studio (FastAPI :8900) → ComfyUI (REST :8188) → SD Model
              │                               │
         POST /generate              POST /prompt → prompt_id
         GET /jobs/{id}              GET /history/{id} → outputs
              │                               │
         asyncio.create_task()       GET /view?filename= → file bytes
         (poll + download)           saved to ./outputs/
```

## Key Patterns

### Async Job Queue

Return job_id immediately, poll in background. Never await generation in HTTP handler.
```python
job_id = str(uuid.uuid4())[:8]
asyncio.create_task(_wait_and_download(job_id, prompt_id))
return {"job_id": job_id, "status": "running"}
```

### Chinese Auto-Translation
SDXL cannot process Chinese. Detect and translate before submission.
```python
if any('\u4e00' <= c <= '\u9fff' for c in prompt):
    result = await optimize_prompt(prompt)
    prompt = result["prompt"]
```

### Workflow Template System
Parameterized JSON workflows with PLACEHOLDER strings. Replace at runtime via `apply_params()`.
CLIPTextEncodeSDXL for SDXL, CLIPTextEncode for SD1.5.

### UI Standards (follow Leonardo/Runway patterns)
- Mode tabs: Image | Video (separate, not mixed)
- Aspect ratio buttons: visual shape buttons, not number inputs
- Negative prompt: collapsed by default
- Advanced: collapsed section
- Style tags: clickable pills
- Output: Preview / Gallery / History tabs
