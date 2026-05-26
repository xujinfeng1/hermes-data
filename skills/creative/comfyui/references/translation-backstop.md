# Chinese Prompt Translation — Backstop Pattern

## The Bug (3x repeat offender)

Chinese text sent directly to SDXL/AnimateDiff → random noise output.
Root cause: translation in API endpoint layer fails silently, Chinese reaches the model.

**Three occurrences in one session:**
1. Endpoint tried to translate → DeepSeek key not in env → silent failure
2. Translation code correct but `event.target` broke JS → user couldn't click optimize
3. Translation in endpoint worked but `generate_animated_video` didn't re-translate

## Correct Pattern

Translation must be the FIRST step INSIDE the engine function, not just the endpoint:

```python
async def generate_video(prompt, ...):
    # ⚠️ Must be inside engine, not just endpoint
    if any('\u4e00' <= c <= '\u9fff' for c in prompt):
        from app.prompt_engine import optimize_prompt
        try:
            result = await optimize_prompt(prompt)
            if result.get("prompt") and not any('\u4e00' <= c <= '\u9fff' for c in result["prompt"]):
                prompt = result["prompt"]
        except:
            pass  # Don't block generation if LLM is down
    # ... submit to ComfyUI
```

## Verification

After every change, check ComfyUI history to verify what prompt reached the model:
```bash
curl -s http://127.0.0.1:8188/history | python3 -c "
import sys,json
d=json.load(sys.stdin)
for pid,h in list(d.items())[-3:]:
    nodes=h.get('prompt',[])[2] if len(h.get('prompt',[]))>2 else {}
    for n in nodes.values():
        if 'CLIP' in n.get('class_type',''):
            t=n.get('inputs',{}).get('text','')
            if t: print(f'{\"CN❌\" if any(\"\\u4e00\"<=c<=\"\\u9fff\" for c in t) else \"EN✅\"}: {t[:80]}')
            break
"
```
