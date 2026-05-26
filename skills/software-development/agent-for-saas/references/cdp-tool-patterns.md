# CDP Tool Patterns

Patterns for building CDP (Customer Data Platform) AI agent tools. Extracted from the `agent-for-saas` project.

## Tool Inventory

| Tool | NL Trigger | Core Pattern |
|------|-----------|-------------|
| segment_users | "圈出女性高消费用户" | Filter dicts by multiple optional params, AND logic |
| list_segments / get_segment_detail | "有哪些预设人群" | Pre-computed segments stored as JSON |
| get_user_profile | "看看陈小明的情况" | Rich object → formatted text dump |
| get_user_events | "他最近做了什么" | Sort by timestamp desc, optional event_type filter |
| list_tags / search_tags | "有哪些标签" | Metadata catalog, keyword search |
| create_marketing_canvas | "给流失用户创建挽回旅程" | Template-driven generation with branching |

## Audience Segmentation (圈人)

```python
def segment_users(gender=None, city=None, level=None, min_age=None, max_age=None,
                  min_total_spent=None, min_orders_30d=None, max_days_inactive=None,
                  tags_include=None, limit=20):
    # Load users from data source
    # For each user, test ALL non-None conditions (AND logic)
    # Return formatted list with key stats
```

Design rules:
- All parameters optional — LLM passes only what the user specified
- AND logic between conditions (OR is too complex for NL→param mapping)
- Return both `total` count and `limit`-trimmed list
- Include key stats in output so LLM can reference them without re-querying

## Marketing Canvas (营销画布)

Template structure:
```python
CANVAS_TEMPLATES = {
    "goal_name": {
        "nodes": [
            {"day": 0, "type": "trigger", "name": "...", "desc": "..."},
            {"day": 1, "type": "message", "name": "...", "desc": "..."},
            {"day": 1, "type": "branch", "name": "条件?",
             "yes": [sub_nodes],
             "no": [sub_nodes]},
            {"day": 3, "type": "ab_test", "name": "A/B测试",
             "a": [sub_nodes], "b": [sub_nodes]},
            {"day": 5, "type": "eval", "name": "效果评估", "desc": "..."},
        ]
    }
}
```

Node types:
- `trigger` — event/time-based activation (not user-facing)
- `message` — channel touchpoint with content
- `branch` — conditional split (yes/no paths)
- `ab_test` — traffic split (a/b groups)
- `eval` — measurement node (not user-facing)

Visualization: recursively flatten nested nodes with tree-drawing characters (`├─`, `└─`).

## Analytics Engine

### Funnel (漏斗分析)

```python
def funnel_analysis(funnel_name="购买转化", steps="", days=7):
    # 1. Load events, filter by time window
    # 2. Build user→{events} map
    # 3. For each step, count users who completed it
    # 4. Compute step-to-step conversion rates
    # 5. Identify bottleneck (max drop-off)
```

### Retention (留存分析)

```python
def retention_analysis(base_event="支付成功", return_event="支付成功"):
    # 1. Find first base_event per user
    # 2. Find all return_events per user after first
    # 3. For each interval (Day1/3/7/14/30), count users with return_event in window
```

### Campaign Overview (活动效果)

```python
def campaign_analysis(days=7):
    # Aggregate: active_users, views, carts, payments, revenue, avg_order_value
    # Funnel: view→cart→pay rates
    # Channel breakdown: payments by channel (wechat/alipay/bank)
```

## System Prompt: Condition Mapping

Essential for NL→param translation. The LLM needs explicit mappings:

```
"高消费" → min_total_spent >= 10000
"活跃/经常买" → min_orders_30d >= 3
"流失/沉默/好久没买" → max_days_inactive <= 21
"一线城市" → city = 北京/上海/广州/深圳
"女性" → gender = "女"
```

Without this mapping table, the LLM guesses parameter values and frequently gets them wrong.
