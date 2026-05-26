# 中国大陆 ComfyUI 安装注意事项

## 网络问题

### HuggingFace 被墙
中国大陆直接访问 huggingface.co 非常慢或超时，`comfy model download` 会挂起。

**解决方案：使用 hf-mirror.com 镜像手动下载**

```bash
# 不要用 comfy model download --url https://huggingface.co/...
# 直接用 curl + 镜像：
cd ~/Documents/comfy/ComfyUI/models/checkpoints
curl -L -o sd_xl_base_1.0.safetensors \
  "https://hf-mirror.com/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors"
```

下载速度通常 3-5 MB/s，SDXL 6.5GB 约需 20-30 分钟。

### 模型 URL 映射

| 原始 HuggingFace URL | 镜像 URL |
|---------------------|---------|
| `https://huggingface.co/...` | `https://hf-mirror.com/...` |

### 检查下载进度

```bash
ls -lh ~/Documents/comfy/ComfyUI/models/checkpoints/sd_xl_base_1.0.safetensors
```

## Apple Silicon 验证

### 硬件检查通过标准
- M1 及以上芯片
- ≥16GB 统一内存（SDXL 推荐 ≥32GB）
- `python3 hardware_check.py --json` 输出 `"verdict": "ok"`

### 安装命令
```bash
# 安装 comfy-cli
pipx install comfy-cli

# 安装 ComfyUI（M 系列芯片）
comfy --skip-prompt tracking disable
comfy --skip-prompt install --m-series

# 启动
comfy launch --background
```

### 验证启动
```bash
curl -s http://127.0.0.1:8188/system_stats | python3 -c "import sys,json; print('OK')"
```

首次启动加载模型需要 10-30 秒，期间 API 返回空响应，轮询等待即可。

## AIGC Studio 集成

ComfyUI 启动后，AIGC Studio 自动检测连接状态：
- `GET /health` 返回 `"comfyui": "online"` 表示已连通
- 通过 REST API 提交工作流、轮询状态、下载结果
- SDXL 文生图约 30-60 秒/张（M1 Max 32GB）
