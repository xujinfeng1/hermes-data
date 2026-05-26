# AIGC Studio Wrapper — Development Rules

Lessons from building a FastAPI wrapper around ComfyUI for Chinese users.

## Rule 1: Check ComfyUI Before Touching Code

When ANY generation fails, diagnose ComfyUI first:

```bash
curl -s http://127.0.0.1:8188/system_stats
curl -s http://127.0.0.1:8188/queue
```

ComfyUI on M1/M2 Macs hangs after hours (MPS memory fragmentation).
Process is alive, HTTP times out. Only fix: kill + relaunch.

## Rule 2: Translation BEFORE apply_params (Two-Layer Backstop)

Chinese-to-English translation MUST execute before `apply_params()` injects
the prompt into the workflow JSON. This bug recurred 4+ times in one session.

**Anti-pattern** (translation after apply_params — Chinese reaches SDXL):
```python
workflow = apply_params(template, params)  # Chinese baked in
# translate here — TOO LATE, workflow already has Chinese
```

**Correct pattern** (translate first, single apply):
```python
if any('\u4e00' <= c <= '\u9fff' for c in params.get("prompt","")):
    result = await llm_translate(params["prompt"])
    if result and no_chinese(result):
        params["prompt"] = result
workflow = apply_params(template, params)  # now English
```

**Two-layer backstop**: implement translation at BOTH layers:
1. **Endpoint layer** (defensive) — translate before calling engine
2. **Engine layer** (mandatory) — translate as FIRST step in generate function

If endpoint translation fails silently, engine catches it.

## Rule 3: Validate JS After Every HTML Patch

When patching an HTML file's inline `<script>`, validate with:
```python
import subprocess
js = html[html.index('<script>')+8:html.index('</script>')]
r = subprocess.run(['node','-e',f'new Function({repr(js)})'],
                   capture_output=True, text=True)
assert not r.stderr
```

Common corruptions from `patch()` operations:
- Orphaned `async` keywords (removed function but left declaration)
- Over-escaped quotes: `\\\\'` instead of `'` in template literals
- Duplicate function declarations (later overrides earlier silently)
- Missing `event` parameter in onclick handlers

## Rule 4: M1 Max 32GB — What Works

| Task | Time | Quality | Verdict |
|------|------|---------|---------|
| SDXL txt2img (30 steps) | 30-60s | Good | ✅ Primary |
| SDXL img2img | 45-90s | Good | ✅ Use |
| AnimateDiff SD1.5 (16f) | 4-8min | Mid | ⚠️ OK |
| HunyuanVideo (17f 480p) | 5-8min | N/A | ❌ Skip |
| HunyuanVideo (49f 720p) | 20+ min | N/A | ❌ Skip |

Focus SDXL image gen for Apple Silicon wrappers. Video is 5-10x slower than RTX 4090.

## Rule 5: China Network — Download Priority

1. ModelScope (`modelscope.cn`): SD1.5, SDXL ✅; motion modules ❌
2. hf-mirror.com: intermittent ~3MB/s
3. ghproxy.net: GitHub only, unreliable
4. Direct HF: blocked, VPN required
5. Terminal doesn't route through system VPN — use browser for manual downloads

Motion modules (AnimateDiff, HunyuanVideo) are NOT on Chinese mirrors.
Browser VPN download is the only reliable path.

## Rule 6: Denoise ONLY for img2img

**Bug:** Frontend sends `denoise: 0.5` in params for ALL modes (txt2img + img2img).
KSampler receives denoise=0.5 for txt2img → only removes 50% noise from pure random
latent → output is snow/noise.

**Frontend fix:**
```js
const params = {prompt, ..., width, height};
if (tpl === 'img2img') params.denoise = parseFloat(denoiseEl?.value || 0.5);
// txt2img: denoise stays at workflow default (1.0)
```

**Backend defense:**
```python
if "denoise" in params and params["denoise"] < 1.0:
    node["inputs"]["denoise"] = params["denoise"]
# Only apply denoise when < 1.0 (img2img). txt2img default (1.0) is left alone.
```

## Rule 7: DeepSeek V4 Pro — reasoning_content

DeepSeek V4 Pro is a reasoning model (like o1). The response has BOTH fields:
- `message.content` — often EMPTY
- `message.reasoning_content` — contains the actual answer

**Fix for ALL LLM API clients:**
```python
msg = data["choices"][0]["message"]
content = msg.get("content", "") or msg.get("reasoning_content", "")
```

Without this, `message.content` is empty string → JSON parse fails silently →
translation and inspiration features appear broken.
