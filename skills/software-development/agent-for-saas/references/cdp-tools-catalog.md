# CDP Agent Tool Reference

## Complete Tool List (37 tools)

### 人群圈选
- segment_users — NL条件筛选用户
- list_segments — 预定义人群列表
- get_segment_detail — 人群详情+用户列表

### 用户画像
- get_user_profile — 360°用户视图
- get_user_events — 行为事件时间线

### 标签管理
- list_tags / search_tags — 标签体系

### 营销画布
- create_marketing_canvas — 7种目标(促首单/促复购/挽回流失/提升客单价/大促预热/弃购挽回/生日营销)

### 数据分析
- funnel_analysis / retention_analysis / campaign_analysis
- attribution_analysis / cohort_analysis

### 预测引擎
- predict_churn / predict_ltv / next_best_action / detect_anomalies

### 自动化
- auto_health_check / auto_execute_campaign / auto_optimize

### 报表
- daily_report / weekly_report

### 知识库
- search_knowledge_base — TF-IDF检索

### 客户旅程
- journey_map — 事件路径分析+桑基图数据

### 任务调度
- schedule_task / list_scheduled_tasks / remove_scheduled_task / run_task_now

### 导出&分享
- generate_chart / export_report / create_share_link

### 多Agent&学习
- orchestrate_agents / learn_from_feedback / get_learned_preferences

### 模型管理
- switch_model / list_models

## Agent Configuration

```python
# models.py
BUILTIN_MODELS = [
    ModelProvider("deepseek", "DeepSeek V4 Pro", "...", "deepseek-v4-pro", "DEEPSEEK_API_KEY", 0.001, 0.002),
    ModelProvider("qwen", "通义千问 Plus", "...", "qwen-plus", "QWEN_API_KEY", 0.0008, 0.002),
    ModelProvider("glm", "智谱 GLM-4", "...", "glm-4", "GLM_API_KEY", 0.001, 0.001),
]
```

## Database Switching

```bash
# Mock (default)
curl -X POST /admin/database -d '{"type":"mock"}'

# MySQL
curl -X POST /admin/database -d '{"type":"mysql","host":"10.0.0.1","user":"root","password":"xxx","database":"cdp"}'

# SQLite
curl -X POST /admin/database -d '{"type":"sqlite","path":"data/cdp.db"}'
```

## External URLs

```
Demo:   http://8.163.113.208:8080/demo
Admin:  http://8.163.113.208:8080/admin
SDK:    <script src="http://8.163.113.208:8080/static/chat-widget.js" data-api-key="KEY"></script>
```
