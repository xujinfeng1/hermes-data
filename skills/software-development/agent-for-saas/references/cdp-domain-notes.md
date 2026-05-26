# CDP Domain Knowledge — Competitive Analysis & Agent Opportunity

> Extracted from session: building a CDP AI Agent product, May 2024.

## CDP Core Feature Modules (Standard)

1. **数据接入与整合**: multi-source data collection (SDK/API/file), real-time/batch pipelines, ETL
2. **OneID / 统一用户画像**: cross-device/channel ID resolution, user attribute merge, 360° view
3. **标签体系**: rule tags (if-then), computed tags (SQL), predictive tags (ML), lifecycle mgmt
4. **人群圈选 (Audience Segmentation)**: visual condition combiner (AND/OR/NOT), behavioral sequence, lookalike, intersection/difference
5. **营销画布 (Marketing Canvas)**: drag-and-drop journey designer, triggers (event/time/attribute), multi-channel (SMS/push/email/WeCom), A/B testing
6. **分析与洞察**: event/funnel/retention analysis, attribution, user path, custom dashboards
7. **数据激活**: audience sync to ad platforms (Douyin/Tencent/Baidu), webhook/API, message channels

## Competitive Landscape — AI Capabilities

| Product | Conversational Agent | NL Segmentation | AI Canvas | AI Analysis | Notes |
|---------|---------------------|-----------------|-----------|-------------|-------|
| Segment (Twilio) | ✗ | Predictive only | ✗ | Predictive | Predictions but no NL |
| mParticle | ✗ | ✗ | ✗ | Basic ML | No conversational |
| Treasure Data | ✗ | ✗ | ✗ | ML models | Data-processing focus |
| BlueConic | ✗ | AI segments | ✗ | AI insights | Pandora ML engine |
| Tealium | ✗ | ✗ | ✗ | Predict | Tag management focus |
| ActionIQ | ✗ | ✗ | ✗ | AI analysis | Finance vertical |
| Braze | ✗ | ✗ | ✗ | AI optimize | Marketing automation (not pure CDP) |
| Salesforce CDP | ✗ | Einstein | Einstein | Einstein | Agentforce exists but locked to SF ecosystem |
| 神策数据 | ✗ | ✗ | ✗ | ML analysis | China CDP leader |
| GrowingIO | ✗ | ✗ | ✗ | Smart analysis | Acquired by 奇点云 |
| 创略科技 | ✗ | ✗ | ✗ | AI predict | Auto/finance focus |
| 诸葛IO | ✗ | ✗ | ✗ | Smart analysis | SMB focused |
| 火山引擎CDP | ✗ | ✗ | ✗ | AI tags | ByteDance ecosystem |
| 个推 | ✗ | ✗ | ✗ | ✗ | Push focused |

**Key finding**: NO CDP product has a true conversational AI agent. This is a market gap.

## Our Agent's Differentiation

Core value proposition: **"用对话代替操作"** — replace drag-and-drop/SQL with natural language.

| Scenario | Traditional CDP | Our Agent |
|----------|----------------|-----------|
| 圈人 | Drag condition combiner / write SQL | "圈出近30天买过3次以上、客单价>200的女性" |
| 创建画布 | Drag nodes, config triggers | "创建新客7天转化旅程：D1优惠券 D3推送 D7短信" |
| 数据分析 | Manual funnel/retention config | "上月加购到支付转化率？环比变化？" |
| 标签管理 | Write SQL for tag definition | "给高价值客户打VIP标签：近90天消费>5000" |
| 策略建议 | Operator experience | "根据上周数据，建议下周重点推哪些人群？" |
| 日报/周报 | Manual dashboard setup | "生成上周营销活动效果报告" |

## Product Positioning

Not building another CDP. Building a **"CDP AI Co-Pilot"** — an independent agent that any CDP SaaS can embed.

**Sell to SaaS companies**: "让你的CDP拥有AI对话能力，降低客户使用门槛"
**End-user value**: "以前要学CDP操作、写SQL、拖画布。现在用中文说就行。"

## Implementation Status (Phase 2)

Project at `/Users/xu/AgentProject/agent-for-saas/`:
- 10 CDP user profiles with full stats (spending, frequency, tags, preferences)
- 34 behavioral events (browse, cart, pay, coupon, review, share, abandon)
- 10 tag definitions across 4 types (attribute, behavioral, computed, predictive)
- 5 predefined audience segments
- 8 agent tools: segment_users, get_user_profile, get_user_events, list_tags, search_tags, list_segments, get_segment_detail, create_marketing_canvas
- Marketing canvas supports 4 goals: 促首单, 促复购, 挽回流失, 提升客单价
