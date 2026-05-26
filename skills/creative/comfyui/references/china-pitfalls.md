# ComfyUI 国内环境与视频工作流实战教训

> 基于 2026-05-24 AIGC Studio 搭建过程中的踩坑记录

## 模型下载策略

| 源 | 可用性 | 适用场景 |
|----|--------|---------|
| hf-mirror.com | 部分可用 | SDXL/SD1.5 基础模型 ✅ |
| modelscope.cn | 可用 | SD1.5 等阿里托管的模型 |
| huggingface.co | 需 VPN | 运动模块、视频模型 |
| ghproxy.net | 不稳定 | 不推荐 |

**关键发现：终端进程不走系统 VPN！** 即使浏览器能访问 HuggingFace，curl/wget/python 请求仍然直连。遇到需要 VPN 下载的模型，让用户浏览器下载后放到指定目录。

## SDXL 工作流铁律

**必须使用 `CLIPTextEncodeSDXL`，不能使用 `CLIPTextEncode`！**

SDXL 有双编码器（CLIP-L + CLIP-G），普通 `CLIPTextEncode` 只用 CLIP-L，导致提示词与画面完全不相关。这是用户在 AIGC Studio 中报告"画面跟提示词毫无关系"的根因。

```json
// ✅ 正确
{"class_type": "CLIPTextEncodeSDXL", "inputs": {
  "text_g": "PROMPT", "text_l": "PROMPT",
  "width": 1024, "height": 1024,
  "target_width": 1024, "target_height": 1024
}}

// ❌ 错误——SDXL 画面与提示词无关
{"class_type": "CLIPTextEncode", "inputs": {"text": "PROMPT"}}
```

## 中文提示词 = 随机噪声

Stable Diffusion 训练数据全是英文。中文提示词直接发给 SD 等于输入随机形状。必须通过 LLM（DeepSeek）翻译成英文后再提交。

**翻译只在端点做不够——必须在引擎内部做双重保障**，否则异步任务中的 prompt 可能跳过翻译。

## AnimateDiff 工作流

### 节点选择

- `AnimateDiffLoaderV2` — **不存在！** ComfyUI 中没有这个类
- `AnimateDiffLoaderV1` — ✅ 存在，配合 SD1.5 使用
- `ADE_ApplyAnimateDiffModelSimple` — ✅ 存在但类型不匹配（输出 M_MODELS，KSampler 要 MODEL）

**正确拓扑（验证通过）：**
```
CheckpointLoaderSimple → AnimateDiffLoaderV1 (model + latent)
                        ├─[0] MODEL → KSampler.model
                        └─[1] LATENT → KSampler.latent_image
EmptyLatentImage → AnimateDiffLoaderV1.latents
CLIPTextEncode → KSampler.positive/negative
KSampler → VAEDecode → VHS_VideoCombine
```

### VHS_VideoCombine 必填参数

```json
{
  "frame_rate": 8,
  "loop_count": 0,
  "filename_prefix": "aigc_anim",
  "format": "image/gif",
  "pingpong": false,        // ← 必须！否则 workflow 验证失败
  "save_output": true,
  "images": ["8", 0]
}
```

### 调试工作流验证错误

通过 ComfyUI API 直接提交测试：
```bash
curl -s -X POST http://127.0.0.1:8188/prompt \
  -H "Content-Type: application/json" \
  -d '{"prompt": <workflow>, "client_id": "test"}'
```
返回的 `node_errors` 字段精确指出类型不匹配和缺失参数。

检查节点输入/输出类型：
```bash
curl -s http://127.0.0.1:8188/object_info/NodeName
```

## ComfyUI 启动问题

- `comfy launch --background` 可能因 sqlalchemy 缺失失败，即使 venv 中已安装
- 直接用 `python main.py` 启动通常能绕过
- 强制 kill 进程后用 `comfy launch` 重启可能报 "already running"，需先确认端口释放

## AI 工具不可用时的用户沟通

不要直接说"工具不可用"或假装没功能——清晰地告诉用户当前状态、为什么不能用、需要什么条件才能用。

例如视频模块下载失败时，应明确说"运动模块所有中国镜像都不通，需要 VPN 浏览器下载，文件放 X 目录"，而不是反复尝试同一个失败路径。

## 工程控制论教训

本节 AIGC Studio 开发中反复违反钱学森工程控制论原则：
1. 没诊断就开方——视频卡住直接改代码，没查 ComfyUI 是否在线
2. 没验证稳定性就加功能——连续加 8 功能，没验证基准功能正常
3. 不吸取教训重复犯错——中文翻译问题修了 4 次才根治
