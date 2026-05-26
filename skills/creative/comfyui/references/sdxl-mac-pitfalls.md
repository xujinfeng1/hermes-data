# SDXL on Mac M1 Max — Performance & Pitfalls

## Performance Verdict

| Task | Model | Time | Viable? |
|------|-------|------|---------|
| 文生图 | SDXL 6.5GB | 30-60s | ✅ 好用 |
| 图生图 | SDXL | 30-60s | ✅ 好用 |
| AnimateDiff 视频 | SD1.5 + 运动模块 1.7GB | 3-5min / 16帧 | ⚠️ 能用，画质一般 |
| HunyuanVideo | 27GB 分体模型 | 15-40min / 49帧 | ❌ 不推荐 |

**结论：** M1 Max 32GB 适合生图，不适合生视频。视频模型太重，MPS 比 CUDA 慢 3-10 倍。

## Installation Pitfalls

### comfy-cli venv vs system Python
`comfy launch --background` 偶尔重建虚拟环境导致 sqlalchemy 等依赖丢失。
**推荐直接启动：**
```bash
cd ~/Documents/comfy/ComfyUI && .venv/bin/python main.py --enable-manager
```

### 长时间运行卡死
ComfyUI 在 M1 Max 上连续运行超过 4-6 小时，MPS 内存碎片化导致 HTTP 无响应。进程在但 `/system_stats` 超时。
**解决：** 定期重启。或用 health-check loop 自动探测。

## SDXL 关键配置

```json
{"class_type":"CLIPTextEncodeSDXL","inputs":{
  "text_g":"PROMPT","text_l":"PROMPT",
  "width":1024,"height":1024,"crop_w":0,"crop_h":0,
  "target_width":1024,"target_height":1024,
  "clip":["4",1]}}
```

- CFG ≥ 7.5，sampler: `dpmpp_2m` + scheduler: `karras`
- steps: 30（标准）/ 20（快速）/ 40（高清）
- 分辨率推荐 1024×1024，最大 ~1536×1536

## 模型下载渠道（中国大陆优先级）

1. `modelscope.cn` — SD1.5/SDXL 通常可用，~5MB/s
2. `hf-mirror.com` — 间歇可用，~3MB/s
3. 浏览器 + VPN — 仅用于运动模块等小众模型
4. `huggingface.co` 直连 — 不可用

## JS 前端验证

每次修改 HTML 后必须语法检查：
```bash
python3 -c "
with open('index.html') as f: html=f.read()
js=html[html.index('<script>')+8:html.index('</script>')]
import subprocess
r=subprocess.run(['node','-e',f'new Function({repr(js)})'],capture_output=True,text=True)
print('ERROR:',r.stderr[:200]) if r.stderr else print('JS OK')
"
```
