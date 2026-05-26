# CDP Agent 工具目录

共 23 个工具，分为 8 个领域：

## 基础 CDP
| 工具 | 功能 |
|------|------|
| segment_users | 条件筛选用户（性别/城市/年龄/等级/消费/标签） |
| list_segments | 列出预定义人群 |
| get_segment_detail | 人群详情 + 用户列表 |
| get_user_profile | 用户 360° 画像 |
| get_user_events | 用户行为事件时间线 |
| list_tags | 标签列表 |
| search_tags | 搜索标签 |

## 营销画布
| 工具 | 功能 |
|------|------|
| create_marketing_canvas | 7 种目标：促首单/促复购/挽回流失/提升客单价/大促预热/弃购挽回/生日营销 |
| auto_execute_campaign | 端到端：分析→圈人→画布→执行方案 |

## 分析引擎
| 工具 | 功能 |
|------|------|
| funnel_analysis | 漏斗分析（浏览→加购→支付） |
| retention_analysis | 留存分析（Day1/3/7/14/30） |
| campaign_analysis | 活动效果总览 + 渠道分布 |
| attribution_analysis | 5 种归因模型对比 |
| cohort_analysis | 同期群分析（月/周分组） |

## 预测引擎
| 工具 | 功能 |
|------|------|
| predict_churn | 流失风险预测 + 干预策略 |
| predict_ltv | 生命周期价值预测 |
| next_best_action | 最优营销动作推荐 |
| detect_anomalies | 指标异常检测 |

## 自动化运营
| 工具 | 功能 |
|------|------|
| auto_health_check | 全店健康巡检 |
| auto_optimize | 智能优化建议（按优先级排序） |

## 报表
| 工具 | 功能 |
|------|------|
| daily_report | 日报（核心指标+趋势+异常） |
| weekly_report | 周报（全维度+用户结构+建议） |

## 知识库
| 工具 | 功能 |
|------|------|
| search_knowledge_base | TF-IDF 检索 CDP 领域知识 |

## API 端点 (28个)
- 对话: /chat, /chat/structured, /chat/stream
- 管理: /admin/keys, /admin/usage, /admin/tenants
- 报表: /reports/generate, /reports, /reports/{file}
- 知识库: /ingest, /kb/stats
- Webhook: /webhooks (CRUD)
- 会话: /sessions, /sessions/{id}
- 健康: /health, /health/detailed
- Demo: /demo
