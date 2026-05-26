# CDP Agent Reference Implementation

Built in one session: 42 tools, 46 APIs, 3-level RBAC, multi-model, multi-tenant.

## Quick Start
```bash
./install.sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8800
# Demo: http://localhost:8800/demo
# Admin: http://localhost:8800/admin
```

## Tool Inventory (42 total)

| Domain | Tools |
|--------|-------|
| Audience | segment_users, list_segments, get_segment_detail |
| Profile | get_user_profile, get_user_events |
| Tags/Canvas | list_tags, search_tags, create_marketing_canvas (7 templates) |
| Analytics | funnel_analysis, retention_analysis, campaign_analysis, attribution_analysis, cohort_analysis |
| Predict | predict_churn, predict_ltv, next_best_action, detect_anomalies |
| Auto | auto_health_check, auto_execute_campaign, auto_optimize |
| Reports | daily_report, weekly_report |
|Journey | journey_map (Sankey data) |
| Schedule | schedule_task, list_scheduled_tasks, remove_scheduled_task, run_task_now |
| Export | generate_chart, export_report, create_share_link |
| Deep | orchestrate_agents, learn_from_feedback, get_learned_preferences |
| Model | switch_model, list_models |
| Platform | execute_code, upload_file, list_uploaded_files, get_trace, list_traces |
| RAG | search_knowledge_base |
| Legacy | search_products, query_order, search_faq |

## Key Architecture Patterns

### `/chat/structured` endpoint
Returns `{reply, structured[], sql, json_filter}`. The `structured` array contains domain objects (SegmentOutput, CanvasOutput, FunnelOutput) extracted from tool calls.

### Canvas template engine
7 templates defined as declarative node trees with trigger/message/branch/ab_test/eval types. Template engine replaces channel placeholders and generates both NL text and structured workflow JSON.

### TF-IDF RAG
No model download needed. Tokenizes Chinese (2-4 char ngrams) + English words, builds TF-IDF index on startup from `data/knowledge_base.json`. `search_knowledge_base` tool queries it.

### Task Scheduler
In-process asyncio loop checking every 30s. Supports `daily 9:00`, `weekly mon 9:00`, `interval 30m` schedules. Task types: report, health_check, campaign. Persists to JSON.

### Data Source Abstraction
`DataSource` ABC → MockSource (JSON files) / SQLSource (MySQL/SQLite). Tools call `get_source().query_users(filters, limit)` transparently. DB switching via `/admin/database` endpoint.

## UI Patterns Worth Reusing

### Chinese IME fix
```javascript
function onKeyDown(e) {
  if (e.key === 'Enter' && !e.shiftKey && !e.isComposing) { send(); }
}
```

### Tool-driven action buttons (not text-matching)
```javascript
const hasTool = (n) => tools.includes(n);
if (hasTool('segment_users')) { /* show canvas button */ }
else if (hasTool('create_marketing_canvas')) { /* show confirm button */ }
```

### Markdown rendering compact mode
```javascript
html.replace(/\n\n+/g, '<br>');
html.replace(/\n/g, '');  // single newlines disappear
```

## Deployment

- frp tunnel: localhost:8800 → ECS:8080 via frpc LaunchAgent
- CDP Agent LaunchAgent: `com.cdp.agent.plist` with KeepAlive+RunAtLoad
- frp pitfall: old sessions take ~90s to expire on frps after kill
