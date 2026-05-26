---
name: agent-for-saas
description: "Build embeddable AI agents for SaaS products — ReAct loop, ToolResult structured output, TF-IDF RAG, tool architecture, RBAC, demo UI, CDP domain patterns, deployment, streaming, and competitive analysis. Covers the full agent development lifecycle from domain research through production deployment."
version: 1.0.0
author: Agent
metadata:
  hermes:
    tags: [agent, saas, cdp, architecture, tool-design]
---

# Agent for SaaS — 构建可嵌入 SaaS 的 AI Agent

本 skill 记录了构建一个可嵌入 CDP/电商 SaaS 的 AI Agent 的核心模式。

## 架构模式

### ToolResult — 双通道输出

Agent 的工具不只返回文本给 LLM 看，还返回结构化数据给 API 消费：

```python
@dataclass
class ToolResult:
    text: str              # LLM 看的 NL 文本
    structured: dict|None  # SaaS 消费的机器数据

def segment_users(...) -> ToolResult:
    return ToolResult(
        text="找到 5 人...",          # 给 LLM
        structured={                  # 给 API
            "type": "segment",
            "sql_where": "gender='女' AND total_spent>=10000",
            "user_ids": [...]
        }
    )
```

Agent 用 `.text` 继续对话，API 端点提取 `.structured` 直接输出 SQL/JSON。

### ReAct 循环 + DeepSeek Function Calling

```
用户消息 → [System Prompt + History + Tools] → DeepSeek API
  → 有 tool_calls? → 执行工具 → 结果追加到 messages → 继续循环
  → 无 tool_calls? → 最终回复
  → 超过 MAX_TOOL_CALLS? → 超时处理
```

DeepSeek 使用 OpenAI 兼容格式。tools 定义用标准 JSON Schema。

### TF-IDF RAG（轻量知识检索）

避免下载大型 embedding 模型（国内网络慢）：
- 使用 sklearn-free 的纯 Python TF-IDF 实现
- 中文分词用 2-4 字滑动窗口 + 正则
- 适合 <1000 篇文档的小型知识库
- 零模型下载，秒启动

### 多工具注册

工具分为独立文件，统一注册：

```
app/tools/
  cdp_audience.py   # 圈人
  cdp_tag.py        # 标签 + 画布
  cdp_analytics.py  # 漏斗/留存/活动
  cdp_predict.py    # 流失/LTV/NBA
  cdp_auto.py       # 巡检/端到端
  cdp_report.py     # 日报/周报
  cdp_rag.py        # 知识检索
  cdp_advanced.py   # 归因/队列
```

每个工具文件包含 1-4 个函数，返回 ToolResult。

## Demo UI 模式

### IME 输入法修复

中文输入法组合文字时按回车不应发送消息：

```javascript
function onKeyDown(e) {
  if (e.key === 'Enter' && !e.shiftKey && !e.isComposing) {
    e.preventDefault();
    send();
  }
}
```

`e.isComposing` 在 IME 激活时为 true。

### 交互式操作确认

在 Agent 回复下方渲染操作按钮：

```javascript
// 根据回复内容智能判断需要哪些操作
function renderActions(msg) {
  if (content.includes('圈')) return '✅ 确认人群 | ✏️ 修改 | 🎯 创建画布';
  if (content.includes('画布')) return '🚀 确认执行 | ✏️ 修改 | 取消';
  // ...
}
```

确认操作使用弹窗（overlay + dialog），执行后替换按钮为状态文本。

### 修改 Agent 输出

就地编辑模式：保留原始内容，修改后存储到会话历史，支持还原。

## 部署模式

### LaunchAgent 持久化（macOS）

```xml
<!-- ~/Library/LaunchAgents/com.cdp.agent.plist -->
<key>KeepAlive</key><true/>
<key>RunAtLoad</key><true/>
```

配合 frp 实现外网访问。注意 frp 的 LaunchAgent 自动重启问题（见 frp-setup skill）。

## RBAC 权限模式

### FastAPI POST body 陷阱

FastAPI 的 `role: str = "admin"` 参数默认从 query string 读取，POST JSON body 里的 `role` 会被忽略，始终使用默认值：

```python
# ❌ 错误：role 从 query string 读，body 里的 role 被忽略
async def create_key(role: str = "admin"): ...

# ✅ 正确：从 request body 读取
async def create_key(request: Request):
    body = await request.json()
    role = body.get("role", "admin")
```

### 权限中间件

```python
def require_role(action: str):
    async def checker(auth=Depends(get_auth)):
        if not check_permission(auth["role"], action):
            raise HTTPException(403, f"权限不足")
        return auth
    return checker
```

权限矩阵：admin(全部) / analyst(chat+analyze+export) / viewer(read)。

## 多模型切换

`app/models.py` 管理模型配置和成本追踪。Agent 动态读取当前模型：

```python
current = model_mgr.get_current()
resp = httpx.post(f"{current.base_url}/chat/completions",
    headers={"Authorization": f"Bearer {model_mgr.get_api_key(current.name)}"},
    json={"model": current.model, ...})
```

## 工程控制论工作流（必须遵守）

系统观→反馈控制→稳定性第一→最优化→可靠性→信息论→鲁棒性→系统辨识→自适应→分级递阶→人机协同。

### 反例

- 视频卡住 → 直接改代码，没先查 ComfyUI 是否在线 ❌
- 连续加 5 功能 → 全部做完才测试，基础功能早已坏 ❌

### 正例

```
问题 → 1.查日志/进程 → 2.定位根因 → 3.最小修复 → 4.验证 → 5.回归基本功能
```

## Demo UI 高级模式

- 桑基图 SVG：journey_map structured 数据用纯 SVG 渲染节点+连线
- 编辑还原：修改后保留 originalContent，支持一键还原
- IME 修复：`e.isComposing` 判断输入法状态

## 项目结构模板

```
agent-for-saas/
├── app/
│   ├── agent.py       # ReAct 循环 + 工具定义 + System Prompt
│   ├── main.py        # FastAPI（鉴权/配额/会话/结构化端点）
│   ├── schemas.py     # Pydantic 模型
│   ├── config.py      # 环境变量配置
│   ├── errors.py      # 错误类型 + LLM 重试
│   ├── auth.py        # API Key + 多租户 + 配额
│   ├── store.py       # SQLite 持久化会话
│   ├── rag.py         # 检索引擎
│   ├── structured.py  # ToolResult + 结构化输出模型
│   ├── webhook.py     # Webhook 管理
│   └── tools/         # 工具实现（按领域分文件）
├── data/              # Mock 数据 + 知识库 + 报表 + 会话 DB
├── demo.html          # 单文件 Demo UI
└── requirements.txt
```

## 开发工作流

构建领域 Agent 的标准流程，适用于任何 SaaS 领域（CDP、电商、CRM 等）：

1. **理解领域**：研究 SaaS 品类核心功能和用户角色
2. **竞品分析优先**：在投入功能开发前，先了解竞争格局。用户重视知道"市场上有什么"再决定做什么
3. **清除运行状态**：重大改动前先停掉运行中的服务，用户偏好干净状态再 pivot
4. **设计 Mock 数据**：足够丰富以展示真实价值 — 10+ 用户画像、30+ 行为事件、覆盖边界 case（VIP 用户、新用户、沉默用户）
5. **构建工具**：一文件一工具类，纯函数、类型参数，按领域概念分组
6. **编写 System Prompt**：角色、工作流、参数映射（NL 短语 → 工具参数），必须指示 Agent 使用工具而非幻觉
7. **FastAPI 包装**：`/chat`、`/chat/stream`、会话管理（in-memory dict，Phase 1），CORS 全开
8. **端到端测试**：使用 `execute_code` + Python `subprocess`（不用 shell 变量，会话 ID 会断裂）

## DeepSeek Streaming 模式

流式输出时 tool_calls 分片到达 SSE chunks。必须积累 delta，不能假设一次性到齐：

```python
async def run_agent_stream(user_message, history):
    messages = [system_prompt, *history, user_message]
    for _ in range(MAX_TOOL_CALLS):
        response = client.post(..., stream=True)
        tool_calls_buffer = {}  # {index: {id, name, args_str}}
        async for chunk in response:
            delta = chunk.choices[0].delta
            if delta.tool_calls:
                for tc in delta.tool_calls:
                    idx = tc.index
                    if idx not in tool_calls_buffer:
                        tool_calls_buffer[idx] = {"id": tc.id or "", "name": tc.function.name or "", "args_str": ""}
                    if tc.function.arguments:
                        tool_calls_buffer[idx]["args_str"] += tc.function.arguments
            elif delta.content:
                yield delta.content  # 流式输出文字
        if tool_calls_buffer:
            # 所有 tool_calls 累积完毕，执行并追加结果
            for tc in sorted(tool_calls_buffer.values()):
                result = execute_tool(tc["name"], json.loads(tc["args_str"]))
                messages.append({"role": "tool", "tool_call_id": tc["id"], "content": result})
            continue  # 循环让 LLM 看到结果
        return  # 无 tool_calls，最终回复已流式输出
```

关键：流式版本需要独立的 `run_agent_stream` 实现，不要试图包装 `run_agent` 做流式。同一轮迭代中不要同时 yield 文本又执行 tool_calls。

## 数据源抽象模式

避免在工具中硬编码 JSON 文件读取，使用 ABC 切换 mock/real DB：

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def query_users(self, filters: dict) -> list[dict]: ...
    @abstractmethod
    def get_user(self, user_id: str) -> dict: ...
    @abstractmethod
    def query_events(self, filters: dict) -> list[dict]: ...

class MockSource(DataSource): ...  # JSON files — Phase 1
class SQLSource(DataSource): ...  # MySQL/SQLite — Phase 3
```

工具通过 `get_source()` 获取当前数据源，切换实现不改变工具签名。

## 多模型管理与回退

```python
class ModelManager:
    def get_current(self) -> ModelProvider
    def set_model(self, name) -> bool
    def record_usage(self, model, tokens_in, tokens_out)
    def try_fallback(self) -> Optional[str]  # 自动切换到备用模型
```

缓存陷阱：删除 `model_config.json` 后才应用默认值变更，否则旧配置在重启后仍存在。

## 验证与测试

开发完成后用 curl 端到端验证：

```bash
curl http://localhost:8800/health

curl -X POST http://localhost:8800/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "查询问题", "session_id": "test"}'

# 多轮测试（用 Python subprocess 而非 shell 变量）：
python3 -c "
import subprocess, json
s = subprocess.run(['curl','-s','-X','POST','http://localhost:8800/chat',
  '-H','Content-Type: application/json','-d',json.dumps({'message':'圈选女性VIP','session_id':'test'})],
  capture_output=True, text=True)
r1 = json.loads(s.stdout)
print(r1['reply'])
"
```

验证 tool_calls_made 确认工具路由正确。验证多轮上下文记忆功能。

## 常见陷阱汇总

- **Tool 返回类型标注**：工具函数声明返回 `str`（LLM function calling 兼容），实际返回 `ToolResult`，由 `_execute_tool` 解包
- **`run_agent_stream` 必须存在**：如果 `main.py` 导入了它，删除会导致服务启动失败
- **状态隔离**：`execute_code` 和 `uvicorn` 是不同的 Python 进程。创建 API key 必须通过 REST API，不能直接写文件
- **API Key 引导问题**：`/admin/keys` 创建端点不能要求鉴权，否则无法创建第一个 key
- **Streaming + tool_calls 不混用**：先积累完整响应，检查 tool_calls，再决定 yield 文本或执行工具
- **会话裁剪**：不限制历史长度会超出模型 token 上限。20 轮是安全默认值

## References 索引

本 skill 的 `references/` 目录包含以下专项知识文件：

| 文件 | 内容 |
|------|------|
| `tool-catalog.md` | CDP Agent 工具目录（37 工具详情） |
| `cdp-tools-catalog.md` | CDP 工具函数签名与参数说明 |
| `cdp-tool-patterns.md` | CDP 领域工具设计模式（圈人/画布/分析） |
| `cdp-agent-patterns.md` | CDP Agent 生产级部署模式 |
| `cdp-agent-reference.md` | CDP Agent 完整实现参考（42 工具、46 API） |
| `cdp-domain-notes.md` | CDP 领域知识：核心功能、竞争格局、Agent 机会 |
| `cdp-competitive-landscape.md` | CDP 市场竞争全景（14 产品分析） |
| `structured-output-pattern.md` | ToolResult 结构化输出详细模式 |
| `rbac-fastapi-pitfall.md` | RBAC + FastAPI POST body 陷阱详解 |
| `fastapi-pitfalls-detail.md` | FastAPI 常见陷阱与解决方案 |
| `deepseek-setup.md` | Hermes 环境中的 DeepSeek API Key 提取 |
| `deployment-configs.md` | LaunchAgent + frp 部署配置示例 |
| `frp-debugging.md` | frp 代理冲突调试流程
