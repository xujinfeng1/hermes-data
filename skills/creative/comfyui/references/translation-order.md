# Translation Order Pitfall

## The Bug

Chinese prompts were reaching SDXL directly, producing unrelated images, despite having auto-translation code.

## Root Cause

Translation happened AFTER `apply_params()` injected the Chinese prompt into the workflow:

```python
# ❌ WRONG ORDER
workflow = apply_params(tmpl["workflow"], params)  # Chinese injected first
# ... translation happens here ...
params["prompt"] = english_prompt
workflow = apply_params(tmpl["workflow"], params)  # re-apply — but too late?
```

Even though the code re-applied after translation, the first `apply_params` already wrote Chinese text into the workflow JSON, and the re-apply didn't always overwrite (node ID/dict mutation issues).

## The Fix

Translate BEFORE the first `apply_params`:

```python
# ✅ CORRECT ORDER
prompt = params.get("prompt", "")
if any('\u4e00' <= c <= '\u9fff' for c in prompt):
    result = await llm_translate(prompt)
    params["prompt"] = result["prompt"]

workflow = apply_params(tmpl["workflow"], params)  # English goes in first time
```

## Detection

Check the ComfyUI history to see what prompt actually reached the model:

```bash
curl -s http://127.0.0.1:8188/history | python3 -c "
import sys,json
d=json.load(sys.stdin)
items=list(d.items())
if items:
    pid,h=items[-1]
    nodes=h.get('prompt',[])[2]
    for nid,node in nodes.items():
        if 'CLIP' in node.get('class_type',''):
            t=node.get('inputs',{}).get('text_g','') or node.get('inputs',{}).get('text','')
            if t:
                cn=any('\u4e00'<=c<='\u9fff' for c in t)
                print(f'{\"❌ CN\" if cn else \"✅ EN\"}: {t[:150]}')
            break
"
```

## Broader Principle

**Translation belongs at the data entry point** — the innermost layer that actually sends data to ComfyUI. Don't rely only on the API endpoint layer; add a backstop inside the engine function itself. This is the same class of bug as the CDP Agent's Chinese translation being fixed by adding translation at the engine level, not just the endpoint level.

Two fallback layers:
1. Endpoint: translate for UX responsiveness (shows translated text to user)
2. Engine: translate as safety net (if endpoint layer fails, engine still protects)
