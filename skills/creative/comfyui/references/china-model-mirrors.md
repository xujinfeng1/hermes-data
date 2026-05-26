# 中国大陆网络环境 — 模型下载指南

## 优先级

1. **ModelScope (modelscope.cn)** — 阿里云镜像，SD1.5/SDXL 常见模型有
   - ~5MB/s，最稳定
   - URL 格式: `https://modelscope.cn/models/<org>/<repo>/resolve/master/<file>`
   - 示例: `https://modelscope.cn/models/AI-ModelScope/stable-diffusion-v1-5/resolve/master/v1-5-pruned-emaonly.safetensors`

2. **hf-mirror.com** — HF 镜像，大部分仓库有
   - ~3MB/s，间歇性不可用
   - URL: 把 `huggingface.co` 换成 `hf-mirror.com`

3. **ghproxy.net** — GitHub 代理
   - 只对 GitHub 仓库有效，经常 404

4. **HuggingFace 直连** — 被墙
   - 超时错误: `Failed to connect to huggingface.co port 443`
   - 需要 VPN

5. **浏览器 + VPN** — 终端 curl/python 不走系统 VPN
   - 让用户在浏览器下载，手动放到模型目录
   - 只有这个方式对大文件（>5GB）或稀有模型可靠

## 特殊模型获取

- **AnimateDiff 运动模块 (mm_sd_v15_v2)**: 所有国内镜像均不可用，必须 VPN 浏览器下载
- **HunyuanVideo 四件套**: 所有在 huggingface.co，VPN 浏览器逐个下载
- **SDXL Base**: ModelScope 可用
- **SD1.5**: ModelScope 可用

## huggingface_hub 库

设置 `HF_ENDPOINT=https://hf-mirror.com` 也经常超时。不如直接 curl 用 mirror。
