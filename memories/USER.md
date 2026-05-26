用户使用中文交流，macOS 15.6.1。偏好：1) 做事前先查行业标准（Leonardo/Runway/Kling等），按成熟的做；2) 严格遵守工程控制论（先诊断再开方，先跑通基本功能再迭代）；3) 连续失败时停下来诊断根因，不要反复 patch；4) 简洁直接回复。M1 Max 32GB 用于本地推理/生图。项目：CDP Agent (agent-for-saas/), AIGC Studio (aigc-studio/)。域名 xujinfeng.top(阿里云)。
§
域名 xujinfeng.top，在阿里云购买，DNS 托管在阿里云（未迁到 Cloudflare）。想通过 Cloudflare Tunnel 在外部访问 Hermes WebUI。
§
做事前先查行业标准（Leonardo/Runway/Kling等），按成熟的做。不要自创交互模式。严格遵守工程控制论：先诊断根因再开方，先跑通基本功能再迭代。连续失败时停下来诊断，不要反复 patch。违规案例：视频卡住连加8功能没查ComfyUI状态(根因MPS内存碎片)；中文翻译 bug 修了3次——每次在端点层加翻译，但异步任务调用引擎函数时 prompt 绕过了翻译。根治方法：翻译逻辑放在数据入口的最内层（引擎函数内部），不只依赖外层端点。
§
AIGC Studio 位于 /Users/xu/AgentProject/aigc-studio/ 端口8900。ComfyUI 在 ~/Documents/comfy/ComfyUI/ 端口8188，直接 `.venv/bin/python main.py --enable-manager` 启动（不要用 comfy launch，它会重建 venv 导致 sqlalchemy 丢失）。已装模型：SDXL(6.5G)、SD1.5(4G)、运动模块 mm_sd_v15_v2.ckpt(1.7G)、HunyuanVideo(27G 未成功运行)。M1 Max 32GB。出问题先查 ComfyUI 是否响应。