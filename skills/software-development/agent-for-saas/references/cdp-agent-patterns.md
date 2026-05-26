# CDP Agent — 关键模式参考

## 数据库抽象层

`app/database.py` — 透明切换 Mock/MySQL/SQLite：

```python
from app.database import get_source, set_source, configure_db

# 工具中统一使用
source = get_source()
users = source.query_users(filters, limit)
user = source.get_user(user_id="U10001")
events = source.query_events(user_id="U10001", days=7)
```

切换方式：
```bash
# Mock（默认）
curl -X POST /admin/database -d '{"type":"mock"}'

# MySQL
curl -X POST /admin/database -d '{"type":"mysql","host":"10.0.0.1","user":"root","password":"xxx","database":"cdp"}'

# SQLite
curl -X POST /admin/database -d '{"type":"sqlite","path":"data/cdp.db"}'
```

DataSource 接口：`query_users(filters, limit)` / `get_user(id, name)` / `query_events(uid, type, days, limit)` / `get_segments()` / `get_tags()` / `get_products()` / `get_orders()` / `get_faq()`

MockSource 读 JSON 文件，SQLSource 用 pymysql/sqlite3。工具代码无需任何修改。

## 多模型切换

`app/models.py` — ModelManager 单例：
- 预定义：DeepSeek V3、通义千问 Plus、智谱 GLM-4
- `models.set_model("qwen")` 切换
- `models.get_current()` 返回当前 ModelProvider
- `models.record_usage(name, tokens_in, tokens_out)` 成本追踪
- API key 从环境变量或 `~/.hermes/.env` 读取
- agent.py 的 `run_agent` 动态读取 `model_mgr.get_current()` 获取 base_url/model/api_key

API 端点：
```bash
GET  /models         # 模型列表+成本
POST /models/switch  # 切换模型 {"model":"deepseek"}
```

## 快速原型流程

本 session 的构建顺序（可复用）：
1. 写 Mock 数据 (JSON) → 立即可测试
2. 写工具函数 → 注册到 `__init__.py` + `agent.py`
3. 写 API 端点 → `main.py`
4. 重启测试 → `launchctl unload/load`
5. 有问题用 `execute_code` 调试，不做无谓的 bash 管道

## frp 调试要点

- Hermes 有 `com.hermes.frpc` LaunchAgent，KeepAlive=true 自动重启
- 改 frpc 配置前：`launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist`
- 改完后：`launchctl load ...`
- 端口 80/8080 通常已开放安全组，8888 及以上大概率被拦截
- frps 90 秒心跳超时后才释放旧代理，切换端口需等待
