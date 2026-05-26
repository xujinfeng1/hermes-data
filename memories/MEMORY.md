信息归档规则：底层逻辑/规律/原则存入 memory（系统状态），操作流程/步骤/工作流存入 skill（控制算法）。记忆回答"是什么、为什么"，技能回答"怎么做"。
§
Node.js v22.20.0 installed at ~/.local/node (darwin-arm64 binary, downloaded from npmmirror.com). Binaries symlinked: ~/.local/bin/{node,npm,npx} → ~/.local/node/bin/{node,npm,npx}. ~/.local/bin is in PATH via ~/.zshrc. Installed because Homebrew not available and GitHub direct access times out on this network. When npm/node needed, ensure PATH includes ~/.local/bin.
§
网络环境在中国大陆，GitHub 直接下载很慢或超时。npm 可用 npmmirror.com，GitHub releases 可用 ghproxy.net 代理。安装工具优先用国内镜像。Homebrew 未安装。自装软件放在 ~/.local/bin/，已加入 PATH。
§
CDP Agent 项目偏好：1) 浅色暖橙主题（主色 #f08050，背景 #faf8f5），Demo 和 Admin 统一配色；2) 聊天内容紧凑排版，行高 1.5，双换行=段落单换行忽略；3) 编辑框最小 200px；4) 操作按钮根据实际调用的工具智能匹配（tool-driven），每条回复 2-3 个相关按钮 + ✏️修改 📋复制；5) 中文输入法问题：onKeyDown 检查 e.isComposing 防止误发送
§
FastAPI RBAC 坑：POST body 里的参数（如 role）不会被 FastAPI 的 query parameter 默认值捕获，始终回退到函数签名的默认值（如 role: str="admin"）。必须用 `body = await request.json()` 读取。影响所有 FastAPI POST 端点。
§
工程控制论干活准则：系统观→反馈控制→稳定性第一→最优化→可靠性→信息论→鲁棒性→系统辨识→自适应→分级递阶→人机协同。违规案例：视频卡住连加8功能没查ComfyUI状态(根因MPS内存碎片)。做功能前先查行业标准(Leonardo/Runway/Kling)，不自创交互模式。
§
M1 Max 32GB 生图能力评估：SDXL ~30-60s/张可用；AnimateDiff 视频跑得动但画质一般速度慢（~5min/16帧）；HunyuanVideo 27GB 模型能加载但太慢（20-40min），不适合实际使用。结论：Mac 本地视频生成天生弱于 NVIDIA CUDA，MPS 慢 3-10 倍。生图/图生图是主力，视频不纠结。
§
AIGC Studio 开发铁律（严禁再犯）：1) 先诊断再开方：任何异常先查 ComfyUI 状态 `/system_stats`+`/queue`，M1 Mac 跑几小时后 MPS 内存碎片化挂掉（进程在但 HTTP 超时），重启可解；2) 中文翻译必须在 apply_params 之前，双重后盾（端点+引擎内部），已反复犯4次；3) 每次 patch HTML 后用 `node -e "new Function(js)"` 验证语法；4) txt2img 绝不能注 denoise 参数（→雪花）；5) DeepSeek V4 Pro: content 空→取 reasoning_content
§
DeepSeek V4 Pro 是推理模型：回答内容在 `reasoning_content` 字段，`content` 字段为空。代码中须 fallback: `content = msg.get("content","") or msg.get("reasoning_content","")`。影响所有调用 DeepSeek API 的场景（优化 prompt、灵感生成等）。
§
腾讯轻量应用服务器（Ubuntu 22.04，4核4GB/40GB SSD/3Mbps 带宽），IP未知。Hermes 已安装在此服务器上。国内网络环境，GitHub 需走 ghproxy.net 代理。LLM API 推荐用 DeepSeek（国内直连延迟低），避免用 OpenAI 等国外 API（延迟高+小带宽卡顿）。